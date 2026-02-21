# TCC 落地实战：优惠券核销的高并发、可回滚与注解式实现

> 1.TCC 常用注解速览
>>
> > 注解是很多 TCC 框架（如 Seata、SOFARPC/Dubbo 的分布式事务扩展）提供的声明式能力，用来把一个接口标记为 TCC 资源，并把 Try/Confirm/Cancel 三阶段方法关联起来，减少样板代码与调用出错概率
> >
> > 在 Seata 中，常见的是：
> >
> > @LocalTCC（标记接口）、@TwoPhaseBusinessAction（标记 Try 并绑定二阶段方法）、@BusinessActionContextParameter（把 Try 参数带入二阶段）
> >
> > 在阿里云 DTX 等框架中，也有同名的 @TwoPhaseBusinessAction 等注解，用法与语义相近。
> >

> 2. Seata 注解与上下文一览
>>
> > 注解与用途
> >
> > @LocalTCC：加在接口上，声明该接口包含 TCC 方法，Seata 会解析为 TCC 资源
> >
> > @TwoPhaseBusinessAction：加在 Try 方法上，声明二阶段方法名（如 commitMethod/rollbackMethod），并给该 TCC 方法起一个全局唯一 name
>>
> > @BusinessActionContextParameter：加在 Try 的参数上，指定参数在 BusinessActionContext 中的键名，供 Confirm/Cancel 读取
> >
> > 上下文与获取
> >
> > BusinessActionContext：TCC 专用上下文，承载 XID、BranchId 以及 Try 阶段通过 @BusinessActionContextParameter 传入的参数；在 Confirm/Cancel 中以方法参数接收
> >
> > RootContext：全局事务的线程级上下文，常用 getXID() 获取全局事务 ID，贯穿 AT/TCC/Saga/XA 等模式
> >
> > 方法签名要点
> >
> > Try 方法第一个参数通常是 BusinessActionContext，后续参数自定义
> >
> > Confirm/Cancel 方法通常仅接收 BusinessActionContext 并返回 boolean（表示二阶段是否成功）
> >

> 3.优惠券核销的注解式 TCC 示例（Seata）
>>
> > 场景约定
> >
> > 券状态：AVAILABLE/LOCKED/USED/CANCELLED；
> >
> > 二阶段需要幂等、防悬挂、空回滚。
> >
> > 全局事务由订单服务开启，优惠券服务作为 TCC 参与者
> >
> > 3.1 定义 TCC 接口（加注解）
> >
```
@LocalTCC
public interface CouponTccAction {
    /**
     * Try：锁定优惠券
     */
    @TwoPhaseBusinessAction(
        name = "couponLock",           // 全局唯一
        commitMethod = "confirm",      // 二阶段确认方法名
        rollbackMethod = "cancel"      // 二阶段取消方法名
    )
    boolean tryLock(
        BusinessActionContext context,
        @BusinessActionContextParameter(paramName = "xid") String xid,
        @BusinessActionContextParameter(paramName = "couponId") Long couponId,
        @BusinessActionContextParameter(paramName = "orderId") String orderId
    );
    /**
     * Confirm：确认核销
     */
    boolean confirm(BusinessActionContext context);
    /**
     * Cancel：取消锁定（退回）
     */
    boolean cancel(BusinessActionContext context);
}
```

>> 3.2 接口实现（含幂等与空回滚/防悬挂要点）
```

@Service
public class CouponTccActionImpl implements CouponTccAction {
    @Autowired private CouponMapper couponMapper;
    @Autowired private CouponFreezeMapper freezeMapper;
    @Override
    @Transactional
    public boolean tryLock(BusinessActionContext context,
                           String xid, Long couponId, String orderId) {
        // 幂等：二阶段已执行则直接成功
        if (freezeMapper.existsByXidAndCouponId(xid, couponId)) {
            return true;
        }
        // 防悬挂：已回滚过，禁止再执行 Try
        if (freezeMapper.isCancelled(xid, couponId)) {
            throw new IllegalStateException("禁止在 Cancel 后执行 Try，xid=" + xid);
        }
        // 业务检查 + 锁定（一阶段本地事务内完成）
        int updated = couponMapper.lockCoupon(xid, couponId, orderId);
        if (updated == 0) {
            throw new RuntimeException("券不可用或已被占用，couponId=" + couponId);
        }
        // 记录冻结流水，便于二阶段与审计
        CouponFreeze freeze = new CouponFreeze();
        freeze.setXid(xid);
        freeze.setCouponId(couponId);
        freeze.setOrderId(orderId);
        freeze.setStatus(FreezeStatus.TRYING.getCode());
        freezeMapper.insert(freeze);
        return true;
    }
    @Override
    public boolean confirm(BusinessActionContext context) {
        String xid = context.getXid();
        Long couponId = Long.valueOf(context.getActionContext("couponId").toString());
        // 幂等：已确认直接成功
        CouponFreeze freeze = freezeMapper.findByXidAndCouponId(xid, couponId);
        if (freeze == null) return true;
        if (freeze.getStatus() == FreezeStatus.CONFIRMED.getCode()) return true;
        if (freeze.getStatus() == FreezeStatus.CANCELLED.getCode()) return false;
        // 确认核销：状态迁移 + 记录使用时间
        int updated = couponMapper.confirmUse(xid, couponId);
        if (updated > 0) {
            freezeMapper.updateStatus(xid, couponId, FreezeStatus.CONFIRMED.getCode());
            return true;
        }
        return false; // 失败由调用方/框架重试
    }
    @Override
    public boolean cancel(BusinessActionContext context) {
        String xid = context.getXid();
        Long couponId = Long.valueOf(context.getActionContext("couponId").toString());
        // 幂等：已回滚直接成功
        CouponFreeze freeze = freezeMapper.findByXidAndCouponId(xid, couponId);
        if (freeze == null) {
            // 空回滚：记录日志并返回成功，避免悬挂
            // log.warn("空回滚，xid={}, couponId={}", xid, couponId);
            return true;
        }
        if (freeze.getStatus() == FreezeStatus.CANCELLED.getCode()) return true;
        // 释放锁定：状态回滚 + 可用时间
        int updated = couponMapper.cancelLock(xid, couponId);
        if (updated > 0) {
            freezeMapper.updateStatus(xid, couponId, FreezeStatus.CANCELLED.getCode());
            return true;
        }
        return false;
    }
}
```

>> 发起方使用（开启全局事务）:
```
@Service
public class OrderService {
    @Autowired private OrderMapper orderMapper;
    @Autowired private CouponTccAction couponTccAction;
    @GlobalTransactional
    public void createOrderWithCoupon(CreateOrderReq req) {
        String orderId = generateOrderId();
        // 1. 创建订单（状态 PENDING）
        Order order = buildOrder(orderId, req);
        orderMapper.insert(order);
        // 2. 锁定优惠券（TCC Try）
        couponTccAction.tryLock(order.getXid(), req.getCouponId(), orderId);
    }
    // 支付成功回调：触发二阶段 Confirm
    public void onPaySuccess(String orderId, String xid) {
        // 查询订单使用的券（略）
        List<Long> couponIds = couponMapper.findCouponIdsByOrderId(orderId);
        for (Long cid : couponIds) {
            couponTccAction.confirm(new BusinessActionContext(xid));
        }
        orderMapper.updateStatus(orderId, OrderStatus.PAID.getCode());
    }
    // 超时/取消：触发二阶段 Cancel
    public void onCancel(String orderId, String xid) {
        List<Long> couponIds = couponMapper.findCouponIdsByOrderId(orderId);
        for (Long cid : couponIds) {
            couponTccAction.cancel(new BusinessActionContext(xid));
        }
        orderMapper.updateStatus(orderId, OrderStatus.CANCELLED.getCode());
    }
}
```

>> 要点回顾:
> >
> > @LocalTCC 放在接口；
> >
> > @TwoPhaseBusinessAction 放在 Try 方法并绑定二阶段方法名
> >
> > @BusinessActionContextParameter 把 Try 参数带入二阶段
> >
> > Confirm/Cancel 方法名需与注解配置一致，且返回 boolean；
> >
> > 二阶段接口要支持幂等与可重试

> 4.常见坑与排查清单
>>
> > 注解位置与签名:
> >
> > > @LocalTCC 必须加在接口上
> > >
> > > @TwoPhaseBusinessAction 必须加在 Try 方法
> > >
> > > Confirm/Cancel 方法通常仅接收 BusinessActionContext 并返回 boolean
>>
> > 上下文取值:
> >
> > >Try 的参数用 @BusinessActionContextParameter 标记，二阶段通过 BusinessActionContext.getActionContext("xxx") 取值；
> > >
> > > 需要全局事务 ID 时用 context.getXid()
>>
> > 幂等、空回滚、防悬挂:
> >
> > > 二阶段方法必须幂等（以 xid+couponId 做状态机判定）；
> > >
> > > 出现 空回滚（Cancel 先于 Try）要能识别并直接成功；
> > >
> > > 出现 悬挂（Cancel 已执行而 Try 后到）要在 Try 端拒绝执行
> >
> > 事务边界:
> > >
> > > Try 阶段要在本地事务内完成检查与锁定；
> > >
> > > 二阶段失败由框架/调用方有限重试；
> > >
> > > 不要吞掉异常，否则会被判定为成功


















































































































































































































































