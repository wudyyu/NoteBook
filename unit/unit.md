## 单位转换
```
package com.sangfor.policy.policy.valueobjects;

import cn.hutool.core.util.EnumUtil;

import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public enum UnitType {
    /**
     * 默认单位
     */
    DEFAULT(0, "-"),

    /**
     * 套
     */
    COVER(1, "套"),

    /**
     * 年
     */
    YEAR(2, "年"),

    /**
     * 次
     */
    TIMES(3, "次", "次数"),

    /**
     * 天
     */
    DAY(4, "天"),

    /**
     * 条
     */
    STRIP(5, "条"),

    /**
     * 台
     */
    UNITS(6, "台"),

    /**
     * 个
     */
    PIECE(7, "个"),

    /**
     * 人天
     */
    MAN_DAY(8, "人天", "人"),

    /**
     * 万条
     */
    MILLION_STRIP(9, "万条"),

    /**
     * GB
     */
    GB(10, "GB", "Gb", "gb"),

    /**
     * MB
     */
    MB(11, "MB", "Mb", "mb"),

    /**
     * 月
     */
    MONTH(12, "月", "月份", "个月"),

    /**
     * 分钟
     */
    MINUTE(13, "分钟", "分"),

    /**
     * 小时
     */
    HOUR(14, "小时", "时"),

    /**
     * Mbps
     */
    MBPS(15, "Mbps"),

    /**
     * 核
     */
    CORE(16, "核"),

    /**
     *
     */
    TB(17, "TB", "Tb", "tb"),

    /**
     * 节点
     */
    NODE(18, "节点"),

    /**
     * 万次
     */
    MILLION_TIMES(19, "万次"),

    /**
     * 每10w行代码
     */
    TEN_MILLION_CODE(20, "每10w行代码"),

    /**
     * 实例
     */
    INSTANCE(21, "实例"),

    /**
     * IP
     */
    IP(22, "IP"),

    /**
     * 秒
     */
    SECOND(23, "秒"),

    /**
     * 域名
     */
    DOMAIN(24, "域名"),

    /**
     * 系统
     */
    SYSTEM(25, "系统"),

    /**
     * 安倍
     */
    A(26, "A"),

    /**
     * vCPU
     */
    VCPU(27, "vCPU"),

    /**
     * 对
     */
    PAIR(28, "对"),

    WEEK(29, "周"),
    ;

    UnitType(Integer value, String name, String... aliasNames) {
        this.value = value;
        this.name = name;
        this.aliasNames = aliasNames == null ? new String[0] : aliasNames;
    }

    public Integer getValue() {
        return value;
    }

    public String getName() {
        return name;
    }

    public String[] getAliasNames() {
        return aliasNames;
    }

    private final Integer value;
    private final String name;

    private final String[] aliasNames;

    /**
     * 本单位转成另一个单位的key
     *
     * @param unitType 单位类型
     * @return key
     */
    public String toUnit(UnitType unitType) {
        return this.value + "_" + unitType.value;
    }

    public List<String> getNames() {
        return Stream.concat(Stream.of(name), Arrays.stream(aliasNames)).collect(Collectors.toList());
    }

    public static List<Integer> needPartUnits() {
        // 套、次、台、个、ip作为单位的虚拟资源，需要拆
        return Stream.of(COVER, TIMES, UNITS, PIECE, IP)
                .map(UnitType::getValue)
                .collect(Collectors.toList());
    }


    public static String getUnitDesc(Integer value) {
        UnitType unitType = EnumUtil.likeValueOf(UnitType.class, value);
        if (unitType == null) {
            return DEFAULT.name;
        }
        return unitType.getName();
    }

    public static UnitType getUnit(String name) {
        return Arrays.stream(values()).filter(u -> u.getNames().contains(name)).findFirst().orElse(null);
    }

}

```

```
import com.google.common.collect.HashBasedTable;
import com.google.common.collect.Table;
import com.sangfor.bill.inside.dto.data.UnitTransferDTO;
import com.sangfor.policy.policy.valueobjects.UnitType;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Map;
import java.util.Set;

/**
 * @ClassName: com.sangfor.bill.acl.UnitTransferService.java
 * @Description: 单位转换类
 *
 * @author: 余亮 32578
 * @date:  2023-09-25 16:08
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class UnitTransferService {
    private static final Table<Integer, Integer, Integer> UNIT_TABLE = HashBasedTable.create();

    static {
        // MB(rowKey) ==> GB(columnKey)  11 => 10  【正负号表示乘除， 小=>大(除法) 大=>小(乘法) 】
        UNIT_TABLE.put(UnitType.MB.getValue(), UnitType.GB.getValue(), -1024);
        // GB(rowKey) ==> MB(columnKey)  10 => 11
        UNIT_TABLE.put(UnitType.GB.getValue(), UnitType.MB.getValue(), 1024);

        // TB ==> GB  17 => 10
        UNIT_TABLE.put(UnitType.TB.getValue(),UnitType.GB.getValue(),1024);
        // GB ==> TB  10 => 17
        UNIT_TABLE.put(UnitType.GB.getValue(),UnitType.TB.getValue(),-1024);

        // TB ==> MB  17 => 11
        UNIT_TABLE.put(UnitType.TB.getValue(),UnitType.MB.getValue(),1024*1024);
        // MB ==> TB  11 => 17
        UNIT_TABLE.put(UnitType.MB.getValue(),UnitType.TB.getValue(),-1024*1024);

        // 年 ==> 月  2 => 12
        UNIT_TABLE.put(UnitType.YEAR.getValue(),UnitType.MONTH.getValue(),12);
        // 月 ==> 年  12 => 2
        UNIT_TABLE.put(UnitType.MONTH.getValue(),UnitType.YEAR.getValue(),-12);

        // 月 ==> 天  12 => 4
        UNIT_TABLE.put(UnitType.MONTH.getValue(),UnitType.DAY.getValue(),30);
        // 天 ==> 月  4 => 12
        UNIT_TABLE.put(UnitType.DAY.getValue(),UnitType.MONTH.getValue(),-30);

        // 月 ==> 周  12 => 29
        UNIT_TABLE.put(UnitType.MONTH.getValue(),UnitType.WEEK.getValue(),4);
        // 周 ==> 月  29 => 12
        UNIT_TABLE.put(UnitType.WEEK.getValue(),UnitType.MONTH.getValue(),-4);

        // 天 ==> 小时 4 => 14
        UNIT_TABLE.put(UnitType.DAY.getValue(),UnitType.HOUR.getValue(),24);
        // 小时 ==> 天  14 => 4
        UNIT_TABLE.put(UnitType.HOUR.getValue(),UnitType.DAY.getValue(),-24);

        // 天 ==> 秒 4 => 23
        UNIT_TABLE.put(UnitType.DAY.getValue(),UnitType.SECOND.getValue(),24*3600);
        // 秒 ==> 天 23 => 4
        UNIT_TABLE.put(UnitType.SECOND.getValue(),UnitType.DAY.getValue(),-24*3600);

        // 天 ==> 周 4 => 29
        UNIT_TABLE.put(UnitType.DAY.getValue(),UnitType.WEEK.getValue(),-7);
        // 周 ==> 天  29 => 4
        UNIT_TABLE.put(UnitType.WEEK.getValue(),UnitType.DAY.getValue(),7);

        // 小时 ==> 周  14 => 29
        UNIT_TABLE.put(UnitType.HOUR.getValue(),UnitType.WEEK.getValue(),-24*7);
        // 周 ==> 小时  29 => 14
        UNIT_TABLE.put(UnitType.WEEK.getValue(),UnitType.HOUR.getValue(),24*7);

        // 小时 ==> 月  14 => 12
        UNIT_TABLE.put(UnitType.HOUR.getValue(),UnitType.MONTH.getValue(),-24*30);
        // 月 ==> 小时  12 => 14
        UNIT_TABLE.put(UnitType.MONTH.getValue(),UnitType.HOUR.getValue(),24*30);

        // 周 ==> 秒  29 => 23
        UNIT_TABLE.put(UnitType.WEEK.getValue(),UnitType.SECOND.getValue(),7*24*3600);
        // 秒 ==> 周  23 => 29
        UNIT_TABLE.put(UnitType.SECOND.getValue(),UnitType.WEEK.getValue(),-7*24*3600);

        // 小时 ==> 分钟 14 => 13
        UNIT_TABLE.put(UnitType.HOUR.getValue(),UnitType.MINUTE.getValue(),60);
        // 分钟 ==> 小时 13 => 14
        UNIT_TABLE.put(UnitType.MINUTE.getValue(),UnitType.HOUR.getValue(),-60);
         // 小时 ==> 秒 14 => 23
        UNIT_TABLE.put(UnitType.HOUR.getValue(),UnitType.SECOND.getValue(),3600);
        // 秒 ==> 小时 23 => 14
        UNIT_TABLE.put(UnitType.SECOND.getValue(),UnitType.HOUR.getValue(),-3600);

        // 分钟 ==> 天 13 => 4
        UNIT_TABLE.put(UnitType.MINUTE.getValue(),UnitType.DAY.getValue(),-60*24);
        // 天 ==> 分钟 4 => 13
        UNIT_TABLE.put(UnitType.DAY.getValue(),UnitType.MINUTE.getValue(),60*24);

        // 分钟 ==> 秒 13 => 23
        UNIT_TABLE.put(UnitType.MINUTE.getValue(),UnitType.SECOND.getValue(),60);
        // 秒 ==> 分钟 23 => 13
        UNIT_TABLE.put(UnitType.SECOND.getValue(),UnitType.MINUTE.getValue(),-60);

        // 万条 ==> 条 9 => 5
        UNIT_TABLE.put(UnitType.MILLION_STRIP.getValue(),UnitType.STRIP.getValue(),10000);
        // 条 ==> 万条 5 => 9
        UNIT_TABLE.put(UnitType.STRIP.getValue(),UnitType.MILLION_STRIP.getValue(),-10000);

        // 万次 ==> 次 19 => 3
        UNIT_TABLE.put(UnitType.MILLION_TIMES.getValue(),UnitType.TIMES.getValue(),10000);
        // 次 ==> 万次 3 => 19
        UNIT_TABLE.put(UnitType.TIMES.getValue(),UnitType.MILLION_TIMES.getValue(),-10000);
    }

    /**
     * @Title: 单位统一转换(例如: MB => GB, TB => GB)
     * @param transfer : 转换前对象
     * @Return com.sangfor.bill.inside.dto.data.UnitTransferDTO
     *
     * @author: 余亮 32578
     * @date:  2023/9/25 16:50
     */
    public UnitTransferDTO unitTransfer(UnitTransferDTO transfer){
        Map<Integer, Map<Integer, Integer>> rowMap = UNIT_TABLE.rowMap();

        if (transfer.getSrcUnit().equals(transfer.getDestUnit())){
            return transfer;
        }

        log.info("Before Unit Transfer,unit={}", transfer);

        if (rowMap.containsKey(transfer.getSrcUnit())) {
            Set<Integer> columnKeySet = rowMap.get(transfer.getSrcUnit()).keySet();
            if (columnKeySet.contains(transfer.getDestUnit())){
                Integer rate = rowMap.get(transfer.getSrcUnit()).get(transfer.getDestUnit());
                UnitTransferDTO unitTransferDTO = UnitTransferDTO.builder().srcUnit(transfer.getSrcUnit()).destUnit(transfer.getDestUnit()).build();
                if (rate < 0){
                    // 除法
                    unitTransferDTO.setNumberInit(transfer.getNumberInit().divide(new BigDecimal(rate).abs(), 8, RoundingMode.HALF_UP));
                    unitTransferDTO.setConvertAmount(transfer.getConvertAmount().multiply(new BigDecimal(rate).abs()));
                } else {
                    // 乘法
                    unitTransferDTO.setConvertAmount(transfer.getConvertAmount().divide(new BigDecimal(rate).abs(), 8, RoundingMode.HALF_UP));
                    unitTransferDTO.setNumberInit(transfer.getNumberInit().multiply(new BigDecimal(rate).abs()));
                }
                return unitTransferDTO;
            }
        }
        log.info("After Unit Transfer,unit={}", transfer);
        return transfer;
    }

}

```