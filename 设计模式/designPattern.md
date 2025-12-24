## 1.策略模式
```
package com.sangfor.order.service.renew.strategy;

import org.springframework.beans.factory.InitializingBean;

public interface RegisterGeneralStrategyBean extends InitializingBean {

    /**
    * @description: 注册策略key
    **/
    String registerStrategyKey();

    /**
    * @description: InitializingBean 实现
    **/
    default void afterPropertiesSet(){
        GeneralStrategyFactory.registerStrategy(registerStrategyKey(), this);
    }
}

```

```
package com.sangfor.order.service.renew.strategy;

import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;

import java.util.HashMap;
import java.util.Map;

@Slf4j
public class GeneralStrategyFactory {
    public static Map<String, Object> strategyMap = new HashMap<>();

    public static <R> void registerStrategy(String key, R strategy) {
        if (StringUtils.isBlank(key)) {
            throw new IllegalArgumentException("register strategy key is blank");
        }
        if (strategyMap.containsKey(key)) {
            throw new IllegalArgumentException("register strategy key is exists");
        }
        strategyMap.put(key, strategy);
    }

    @SuppressWarnings("unchecked")
    public static <R> R getStrategy(String key) {
        return (R) strategyMap.get(key);
    }
}

```

```
package com.sangfor.order.service.renew.strategy.impl;

import com.sangfor.order.service.renew.strategy.RegisterGeneralStrategyBean;
import com.sangfor.orderapi.enums.EffectiveType;
import com.sangfor.orderapi.pojo.dto.renew.RenewListChangeDTO;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@RequiredArgsConstructor
public abstract class AbstractRenewStrategy implements RegisterGeneralStrategyBean {

    /**
    * @description: 生效类型
    * @author: 余亮 32578
    * @date: 2025/10/14 11:18
    * @methods: getEffectiveType
    **/
    public abstract EffectiveType getEffectiveType();

    public abstract void executeChange(RenewListChangeDTO renewListChangeDTO);

    public static String generateRegisterKey(String effectType) {
        return String.join("_", "RENEW", effectType);
    }

    @Override
    public String registerStrategyKey() {
        return generateRegisterKey(String.valueOf(getEffectiveType().getCode()));
    }
}

```


```
package com.sangfor.order.service.renew.strategy.impl;

import com.sangfor.orderapi.enums.EffectiveType;
import com.sangfor.orderapi.pojo.dto.renew.RenewListChangeDTO;
import org.springframework.stereotype.Component;

/**
* @description: 新续费周期策略
* @author: 余亮 32578
* @date: 2025/10/14 11:25
*/
@Component
public class RenewEffectiveLiftCycleStrategy extends AbstractRenewStrategy{
    @Override
    public EffectiveType getEffectiveType() {
        return EffectiveType.EFFECTIVE_TYPE_NEW_LIFECYCLE;
    }

    @Override
    protected void executeChange(RenewListChangeDTO renewListChangeDTO) {

    }

}

```

```
package com.sangfor.order.service.renew.strategy.impl;

import com.sangfor.orderapi.enums.EffectiveType;
import com.sangfor.orderapi.pojo.dto.renew.RenewListChangeDTO;
import org.springframework.stereotype.Component;

/**
* @description: 指定时间策略
* @author: 余亮 32578
* @date: 2025/10/14 11:25
*/
@Component
public class RenewEffectiveSpecifiedStrategy extends AbstractRenewStrategy{

    @Override
    public EffectiveType getEffectiveType() {
        return EffectiveType.EFFECTIVE_TYPE_SPECIFIED;
    }

    @Override
    protected void executeChange(RenewListChangeDTO renewListChangeDTO) {

    }
}

```

## 2.策略+模板

```
package com.sangfor.coupon.service.strategy.flow;

/**
 * @Description: 账单核销模板
 * @ClassName: com.sangfor.coupon.service.strategy.flow.BillChargeHandler.java
 *
 * @author: 余亮 32578
 * @date:  2024-06-04 15:13
 */
public interface BillChargeTemplate<T, R>  {
    /**
     * 核销前置方法
     * @param t : 核销对象
     * @return boolean
     *
     * @author: 余亮 32578
     * @date:  2024/6/4 15:18
     */
    boolean before(T t);

    /**
     * 核销后置方法
     * @param t : 核销对象
     * @param r : 核销返回
     *
     * @author: 余亮 32578
     * @date:  2024/6/4 15:17
     */
    void after(T t, R r);

    /**
     * 账单核销
     * @param t : 核销对象
     *
     * @author: 余亮 32578
     * @date:  2024/6/4 15:17
     */
    void recharge(T t);

    /**
     * 账单核销处理函数
     * @param t : 核销对象
     * @return R 核销返回
     *
     * @author: 余亮 32578
     * @date:  2024/6/4 19:52
     */
    R handle(T t);
}

```

```
package com.sangfor.coupon.service.strategy.flow;

import lombok.extern.slf4j.Slf4j;

/**
 * @Description: 抽象账单核销处理器
 * @ClassName: com.sangfor.coupon.service.strategy.frozen.AbstractBillFlowChargeHandler.java
 *
 * @author: 余亮 32578
 * @date:  2024-06-04 15:30
 */
@Slf4j
public abstract class AbstractBillChargeHandler<T extends Object, R extends Object> implements BillChargeTemplate<T, R>{

    @Override
    public R handle(T t){
        if (!before(t)){
            log.error("Execute Bill Recharge Before Failed!");
            return null;
        }

        recharge(t);

        return null;
    }
}

```

```
package com.sangfor.coupon.service.strategy.flow;

import com.sangfor.bill.inside.constant.CommonConstant;
import com.sangfor.platform.enums.CommonErrorCodeEnum;
import com.sangfor.platform.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang3.StringUtils;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.stereotype.Service;
import org.springframework.util.ObjectUtils;
import java.util.HashMap;
import java.util.Map;

/**
 * @Description: 账单核销策略工厂类
 * @ClassName: com.sangfor.coupon.service.strategy.frozen.BillChargeStrategyFactory.java
 *
 * @author: 余亮 32578
 * @date:  2024-06-04 17:33
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class BillChargeStrategyFactory implements InitializingBean {

    /**
     * 存放策略Map
     */
    public static Map<String, AbstractBillChargeHandler> strategyMap = new HashMap<>();

    /**
     * 包年包月策略类型
     */
    public static final String PRE_PAID_KEY = "pre_paid";

    /**
     * 按量计费策略类型
     */
    public static final String POST_PAID_KEY = "post_paid";

    public static final String FREEZE = "freeze";

    /**
     * 按量计费策略
     */
    private final PostPaidBillChargeHandler postPaidBillCharge;

    /**
     *包年包月策略
     */
    private final PrePaidBillChargeHandler prePaidBillCharge;

    /**
     * 策略Map对应的Key
     *
     * @param key 策略Key
     * @return com.sangfor.charge.service.startegy.AbstractBillFlowHandler
     * @author 余亮 32578
     */
    public AbstractBillChargeHandler getStrategy(String key) {
        AbstractBillChargeHandler strategy = strategyMap.get(key);
        if (ObjectUtils.isEmpty(strategy)) {
            throw new BusinessException(CommonErrorCodeEnum.COMMON_MESSAGE, "bill.exception.strategy.isEmpty");
        }
        return strategy;
    }

    /**
     * 根据冻结编号返回冻结类型
     *
     * @param freezeId 冻结编号
     * @return java.lang.String
     * @author 余亮 32578
     */
    public static String getFreezeType(String freezeId) {
        return StringUtils.isBlank(freezeId) ? POST_PAID_KEY : PRE_PAID_KEY;
    }

    /**
     * 获取策略Key
     *
     * @param freezeId 冻结编号
     * @return java.lang.String
     * @author 余亮 32578
     * @date 2023/5/15
     */
    public static String getStrategyKey(String freezeId){
        return String.join(CommonConstant.UNDERLINE, BillChargeStrategyFactory.FREEZE,
                BillChargeStrategyFactory.getFreezeType(freezeId));
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        strategyMap.put(String.join(CommonConstant.UNDERLINE, FREEZE, POST_PAID_KEY), postPaidBillCharge);
        strategyMap.put(String.join(CommonConstant.UNDERLINE, FREEZE, PRE_PAID_KEY), prePaidBillCharge);
    }
}

```

```
package com.sangfor.coupon.service.strategy.flow;

import com.sangfor.bill.service.BillRechargeAuxiliaryService;
import com.sangfor.charge.pojo.RechargeDTO;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

/**
 * @Description: 账单核销处理器
 * @ClassName: com.sangfor.coupon.service.strategy.frozen.BillFlowCharge.java
 *
 * @author: 余亮 32578
 * @date:  2024-06-04 15:44
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class BillChargeHandler extends AbstractBillChargeHandler<RechargeDTO,Object> {

    /**
     * 账单核销辅助Service
     */
    private final BillRechargeAuxiliaryService rechargeAuxiliaryService;

    @Override
    public boolean before(RechargeDTO rechargeDTO) {
        boolean result;
        // 检查账单是否已经核销
        result = rechargeAuxiliaryService.checkBillRecharge(rechargeDTO);
        if (!result){
            return Boolean.FALSE;
        }
        // 补充物料信息
        rechargeAuxiliaryService.addMaterialIntoBillCharge(rechargeDTO);
        // 重置账单信息
        rechargeAuxiliaryService.resetBillFlow(rechargeDTO);

        return Boolean.TRUE;
    }

    @Override
    public void after(RechargeDTO rechargeDTO, Object o) {

    }

    @Override
    public void recharge(RechargeDTO rechargeDTO) {

    }

}

```

```
package com.sangfor.coupon.service.strategy.flow;

import com.sangfor.bill.aggregates.BillFlow;
import com.sangfor.bill.enums.BillPayType;
import com.sangfor.bill.factory.BillFlowFactory;
import com.sangfor.bill.inside.constant.CommonConstant;
import com.sangfor.bill.inside.dto.data.CreditStatusCheckDTO;
import com.sangfor.bill.service.BillRechargeAuxiliaryService;
import com.sangfor.charge.pojo.BillFlowDTO;
import com.sangfor.charge.pojo.RechargeDTO;
import com.sangfor.config.BillRedisKey;
import com.sangfor.config.RefundConfiguration;
import com.sangfor.coupon.ability.impl.IBillFlowDomainService;
import com.sangfor.coupon.assembler.PaymentAssembler;
import com.sangfor.enums.AccountType;
import com.sangfor.enums.BusinessType;
import com.sangfor.payment.CreditAccount;
import com.sangfor.payment.PaymentAuxiliaryService;
import com.sangfor.payment.WalletAccount;
import com.sangfor.payment.WalletPayment;
import com.sangfor.payment.data.PaymentRequest;
import com.sangfor.platform.enums.CommonErrorCodeEnum;
import com.sangfor.platform.exception.service.ServiceException;
import com.sangfor.platform.log.enums.common.EnabledEnum;
import com.sangfor.platform.redis.client.RedisClient;
import com.sangfor.platform.redis.lock.RedisLock;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.transaction.support.TransactionTemplate;
import java.math.BigDecimal;

/**
 * @Description: 按量计费账单核销
 * @ClassName: com.sangfor.coupon.service.strategy.frozen.PostPaidBillCharge.java
 *
 * @author: 余亮 32578
 * @date:  2024-06-04 17:11
 */
@Slf4j
@Component
public class PostPaidBillChargeHandler extends BillChargeHandler {

    private final RedisClient redisClient;

    private final TransactionTemplate transactionTemplate;

    private final WalletAccount walletAccount;

    private final RefundConfiguration refundConfiguration;

    private final WalletPayment walletPayment;

    private final CreditAccount creditAccount;

    private final IBillFlowDomainService billFlowDomainService;

    public PostPaidBillChargeHandler(BillRechargeAuxiliaryService rechargeAuxiliaryService,
                                     CreditAccount creditAccount, RedisClient redisClient,
                                     TransactionTemplate transactionTemplate,
                                     WalletAccount walletAccount, WalletPayment walletPayment,
                                     RefundConfiguration refundConfiguration,
                                     IBillFlowDomainService billFlowDomainService) {
        super(rechargeAuxiliaryService);
        this.redisClient = redisClient;
        this.walletAccount = walletAccount;
        this.creditAccount = creditAccount;
        this.transactionTemplate = transactionTemplate;
        this.walletPayment = walletPayment;
        this.refundConfiguration = refundConfiguration;
        this.billFlowDomainService = billFlowDomainService;
    }

    @Override
    public void recharge(RechargeDTO rechargeDTO) {
        log.info("Enter PostPaidStrategy recharge, rechargeDTO={}", rechargeDTO);
        RedisLock rLock = null;
        BillFlowDTO bf = rechargeDTO.getBillFlow();
        try {
            rLock = redisClient.getLock(BillRedisKey.getBatchNoKey(bf.getBatchNo()));
            transactionTemplate.executeWithoutResult(s -> {
                // 1.钱包支付
                cashPay(rechargeDTO);

                // 2.核销账单
                chargeBillFlow(rechargeDTO);

                // 3.负数&退款账单退款
                refund(rechargeDTO);
            });
        } catch (Exception e) {
            log.error("Recharge PostPaid Bill Occurs An Error", e);
            throw new ServiceException(CommonErrorCodeEnum.COMMON_MESSAGE, e.getMessage());
        } finally {
            if (null != rLock) {
                rLock.unlock();
            }
        }
    }

    /**
     * 钱包支付
     * 没有冻结编号场景：1.线上修复的账单(只有包年包月账单，没有按量计费账单，没有退费账单)  2.退款账单  3.按量计费账单
     *
     * @param rechargeDTO 核销对象
     * @author 余亮 32578
     */
    public void cashPay(RechargeDTO rechargeDTO) {
        BigDecimal amount = new BigDecimal(rechargeDTO.getBillFlow().getRequireAmount());
        BillFlowDTO bf = rechargeDTO.getBillFlow();

        // 满足按量计费支付条件
        PaymentRequest paymentRequest = PaymentAssembler.makePaymentRequest(
                bf.getProjectId(), bf.getOwnerProjectId(), bf.getBatchNo(),
                bf.getChargeType(), amount.abs(), AccountType.POST_PAID
        );

        AccountType accountType = PaymentAuxiliaryService.routeAccountType(rechargeDTO.getBillFlow().getProjectType(), paymentRequest.getBusinessType());
        boolean canCreditPay = canCreditPay(bf);
        // 设置后付费标志（信用支付）
        rechargeDTO.setPayType(canCreditPay ? BusinessType.POST_PAID.getCode() : BusinessType.PRE_PAID.getCode());

        // 是否满足按量计费支付条件
        if (!bf.canPostPaidPay()){
            return;
        }

        paymentRequest.setAccountType(canCreditPay ? AccountType.CREDIT_BALANCE : accountType);
        paymentRequest.setSupportInterceptors(PaymentAuxiliaryService.getSupportInterceptors(
                rechargeDTO.getBillFlow().getProjectType(), paymentRequest.getBusinessType(), canCreditPay));
        walletPayment.pay(paymentRequest);
    }
     /**
     * 核销账单
     * 服务商 && 信用支付（后付款），账单状态：不核销
     *
     * @param rechargeDTO 核销对象
     * @author 余亮 32578
     */
    public void chargeBillFlow(RechargeDTO rechargeDTO) {
        BillFlowDTO billFlowDTO = rechargeDTO.getBillFlow();

        // 如果自动退款关闭，不更新核销状态和实付金额
        if (billFlowDTO.belongRefund() && !refundConfiguration.isAutoRefund()) {
            return;
        }

        BillFlow bf = BillFlowFactory.INSTANCE.createBillFlow(billFlowDTO);
        bf.makePostPaidWithPayType(new BigDecimal(billFlowDTO.getRequireAmount()), BillPayType.getByCode(rechargeDTO.getPayType()));

        log.info("Enter chargeBillFlow Function,Execute An BillFlow={}", bf);
        billFlowDomainService.modifyBillFlow(bf);
    }

    /**
     * 按量计费退款
     *
     * @param rechargeDTO 核销对象
     * @author 余亮 32578
     * @date 2023/5/25
     */
    public void refund(RechargeDTO rechargeDTO) {
        BillFlowDTO bf = rechargeDTO.getBillFlow();
        // 如果自动退款关闭，直接返回
        if (!refundConfiguration.isAutoRefund()) {
            return;
        }

        BigDecimal amount = new BigDecimal(bf.getRequireAmount());
        // （非退款||正数）订单不退款
        if (!bf.belongRefund() || BigDecimal.ZERO.compareTo(amount) <= CommonConstant.ZERO) {
            return;
        }

        PaymentRequest refundCmd = PaymentAssembler.makePaymentRequest(
                bf.getProjectId(), bf.getOwnerProjectId(), bf.getBatchNo(), bf.getChargeType(),
                new BigDecimal(bf.getRequireAmount()).abs(), AccountType.COMMON);
        walletAccount.walletRefund(refundCmd);
    }

    /**
     * 检查当前账单是否可以信用支付
     * 1.查看租户/渠道账户 是否支持信用支付
     * 2.查看信用支付额度 是否足够
     * @param bf : 账单流水
     * @return boolean
     *
     * @author: 余亮 32578
     * @date:  2024/4/25 16:06
     */
    public boolean canCreditPay(BillFlowDTO bf) {
        EnabledEnum statusType = creditAccount.getCreditStatus(CreditStatusCheckDTO.builder()
                .projectId(bf.getProjectId())
                .ownerProjectId(bf.getOwnerProjectId())
                .projectType(bf.getProjectType()).build());
        return EnabledEnum.ENABLE == statusType &&
                walletAccount.checkBalanceCanPay(bf.getProjectId(), AccountType.CREDIT_BALANCE, new BigDecimal(bf.getRequireAmount()));
    }
}

```

```
package com.sangfor.coupon.service.strategy.flow;

import com.sangfor.bill.entities.InstanceFrozenEntity;
import com.sangfor.bill.enums.BillPayType;
import com.sangfor.bill.gateway.IInstanceFrozenRepository;
import com.sangfor.bill.service.BillRechargeAuxiliaryService;
import com.sangfor.charge.pojo.BillFlowDTO;
import com.sangfor.charge.pojo.RechargeDTO;
import com.sangfor.config.BillRedisKey;
import com.sangfor.config.RefundConfiguration;
import com.sangfor.coupon.service.strategy.recharge.AbstractBillRechargeHandler;
import com.sangfor.coupon.service.strategy.recharge.BillRechargeStrategyFactory;
import com.sangfor.enums.BusinessType;
import com.sangfor.exception.BillErrorCodeEnum;
import com.sangfor.manager.ManualTransactionManager;
import com.sangfor.platform.asserts.AssertHelper;
import com.sangfor.platform.enums.CommonErrorCodeEnum;
import com.sangfor.platform.exception.service.ServiceException;
import com.sangfor.platform.redis.client.RedisClient;
import com.sangfor.platform.redis.lock.RedisLock;
import com.sangfor.platform.util.string.StringUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionStatus;
import java.util.List;

/**
 * @Description: 包年包月账单核销
 * @ClassName: com.sangfor.coupon.service.strategy.frozen.PrePaidBillCharge.java
 *
 * @author: 余亮 32578
 * @date:  2024-06-04 17:12
 */
@Slf4j
@Component
public class PrePaidBillChargeHandler extends BillChargeHandler {

    private final RedisClient redisClient;

    /**
     * 核销策略工厂
     */
    private final BillRechargeStrategyFactory rechargeStrategyFactory;

    private final ManualTransactionManager transactionManager;

    private final IInstanceFrozenRepository instanceFrozenRepository;

    private final RefundConfiguration refundConfiguration;

    public PrePaidBillChargeHandler(BillRechargeAuxiliaryService rechargeAuxiliaryService,
                                    IInstanceFrozenRepository instanceFrozenRepository,
                                    BillRechargeStrategyFactory rechargeStrategyFactory,
                                    ManualTransactionManager transactionManager,
                                    RedisClient redisClient,RefundConfiguration refundConfiguration) {
        super(rechargeAuxiliaryService);
        this.redisClient = redisClient;
        this.rechargeStrategyFactory = rechargeStrategyFactory;
        this.transactionManager = transactionManager;
        this.instanceFrozenRepository = instanceFrozenRepository;
        this.refundConfiguration = refundConfiguration;
    }
     @Override
    public void recharge(RechargeDTO rechargeDTO) {
        log.debug("Enter PrePaidStrategy recharge, rechargeDTO={}", rechargeDTO);

        RedisLock rLock = null;
        BillFlowDTO bf = rechargeDTO.getBillFlow();
        TransactionStatus transactionStatus = null;
        try {
            rLock = redisClient.getLock(BillRedisKey.getFreezeKey(bf.getFreezeId()));

            // 查询冻结记录
            List<InstanceFrozenEntity> frozenEntities = instanceFrozenRepository.queryByFreezeId(bf.getFreezeId());
            AssertHelper.notEmpty(frozenEntities, BillErrorCodeEnum.COUPON_FROZEN_NOT_EXISTS);

            // 同一批冻结记录的支付类型一定是相同的(取第一个)
            String payTypeCode = frozenEntities.stream().map(InstanceFrozenEntity::getPayType)
                    .filter(StringUtil::isNotBlank).findFirst().orElse(BillPayType.PRE_PAID.getCode());
            BillPayType payType = BillPayType.getByCode(payTypeCode);

            // rechargeDTO补录字段(开了信用开关则表示 后付费)
            addRechargeInfo(rechargeDTO, payType);
            log.debug("Add PayType Into rechargeDTO={}", rechargeDTO);

            // 获取核销处理器
            AbstractBillRechargeHandler rechargeHandler = rechargeStrategyFactory.getRechargeStrategy(rechargeDTO.getPayType());
            transactionStatus = transactionManager.begin(TransactionDefinition.PROPAGATION_REQUIRES_NEW, TransactionDefinition.ISOLATION_READ_COMMITTED);

            if (bf.belongRefund()) {
                // 如果自动退款关闭，直接返回
                if (!refundConfiguration.isAutoRefund()) {
                    log.debug("Handle Refund Bill Do Nothing");
                } else {
                    // 退款账单
                    rechargeHandler.refundBillRecharge(rechargeDTO, frozenEntities);
                }
            } else {
                // 处理非退款账单
                rechargeHandler.billRecharge(rechargeDTO, frozenEntities);
            }

            transactionManager.commit(transactionStatus);
        } catch (Exception e) {
            transactionManager.rollback(transactionStatus);
            log.error("Recharge PrePaid Bill Occurs An Error", e);
            throw new ServiceException(CommonErrorCodeEnum.COMMON_MESSAGE, e.getMessage());
        } finally {
            if (null != rLock) {
                rLock.unlock();
            }
        }
    }

    /**
     * 核销对象补录数据
     *
     * @param rechargeDTO 核销对象
     * @param payType 支付类型(pre_paid:预付费 post_paid:后付费)
     * @author 余亮 32578
     * @date 2023/8/8
     */
    private void addRechargeInfo(RechargeDTO rechargeDTO ,BillPayType payType){
        if (BillPayType.POST_PAID == payType){
            rechargeDTO.setPayType(BusinessType.POST_PAID.getCode());
        } else {
            rechargeDTO.setPayType(BusinessType.PRE_PAID.getCode());
        }
    }
}

```

```
/**
 * 核销执行器
 * @author 余亮
 * @date 2023/4/11 16:05
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class RechargeCmdExe {

    /**
     * 核销策略工厂类
     */
    private final BillChargeStrategyFactory billFlowStrategyFactory;

    /**
     * 核销提货券执行器
     * @param rechargeDTO 券核销对象
     * @return java.lang.Boolean
     * @author 余亮 32578
     * @date 2023/4/18
     */
    public Boolean execute(RechargeDTO rechargeDTO){
        log.debug("Execute An RechargeDTO After Credit, rechargeDTO={}", rechargeDTO);
        AbstractBillChargeHandler billFlowHandler = billFlowStrategyFactory.getStrategy(
                BillChargeStrategyFactory.getStrategyKey(rechargeDTO.getBillFlow().getFreezeId()));
        billFlowHandler.handle(rechargeDTO);
        return Boolean.TRUE;
    }

}
```
##