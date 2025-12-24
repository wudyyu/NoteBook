##  DB
> 非 db_ 前缀，按照业务命名，不要超过16个字符（尽量简洁明了）

## 数据表
> 1.禁止建表语句带有容易导致结果歧义/不确定的语法IF Exists.

> 2.禁止建表语句带有Create Table Like语法(容易导致canal订阅发生异常).

> 3.默认表的存储引擎必须是Innodb.

> 4.建表语句必须包含comment备注说明.

> 5.表的编码必须是utf8或utf8mb4.

> 6.表名称不能以如下预留字符开头或结尾:[,new, old, gho, del, ghc, t, v_, c_].

> 7.表名长度不能超过60.

> 8.表名包含了时间格式[线上场景下请放弃基于时间的切表操作].

> 9.表只能以小写字符进行命名.

> 10.禁止Rename Table操作.

##  字段列
> 1.表必须存在主键列: id bigint NOT NULL AUTO_INCREMENT COMMENT '主键'.

> 2.表列数不能超过60(太多的列可能意味着业务可能需要拆分/解耦).

> 3.字段长度不能超过32(丑陋的命名!).

> 4.自增列不能设置默认值,且名称必须是id.

> 5.只能有一个auto_increment属性.

> 6.字段只能以小写字符进行命名.

> 7.字段编码必须是utf8或utf8mb4.

> 8.字段必须要有comment注释.

> 9.字段是timestamp类型必须指定默认值(建议为:CURRENT_TIMESTAMP).

> 10.列不支持bit类型.

> 11.不得出现存在重复列.

> 12.禁止Drop Column.

> 13.禁止Rename Column操作.

> 14.大量的varchar[大字段]会影响性能请优化(varchar(N<200)视为正常).

> 15.单表大字段(tinytext~text)不得超过2个.

> 16.必须包含内建字段created_at, 格式:created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间'.

> 17.内建字段created_at统一约定放在表的最后一列的位置.

> 18.内建字段:created_at必须建立索引ix_created_at.

> 19.必须包含内建字段updated_at, 格式:updated_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间'.

> 20.内建字段update_at统一约定放在表的倒数第二列的位置.

> 21.内建字段:updated_at必须建立索引ix_updated_at.

> 22.字段Modify操作时不得丢失默认值,不得丢失精度,不得丢失Comment描述信息.

> 23.禁止使用大类型如: longtext, mediumtext, blob, mediumblob, longblob

> 24.禁止使用预留字段

##  索引
> 1.主键列必须是[id]命名,类型必须是bigint,必须是auto_increment属性,主键索引不能包含多个列.

> 2.禁止使用外键.

> 3.禁止使用全文索引.

> 4.普通索引命名格式:ix_[列名].

> 5.唯一索引命名格式:uk_[列名].

> 6.唯一索引包含的列必须为Not Null属性.

> 7.不被支持的语法/语法已过期:create index xx on xx,请使用alter table xx add index...语法代替!.

> 8.不被支持的语法/语法已过期:drop index xx on xx,请使用alter table xx drop index...语法代替!.

> 9.索引数超过字段数的1/3索引使用不合理,请联系DBA排查.

> 10.索引中列需存在,且不能是无索引意义的枚举/布尔类型(若疑似会提示).

> 11.索引中列宽度不得大于128字符[谨慎使用大字段做索引].

> 12.组合索引:列数不能超过3个,不得出现重复列,不能包含主键索引列,不能包含唯一索引列.

> 13.复合索引时间类型只能出现在索引尾部.

> 14.索引名称,不得存在重复,需要删除的冗余索引.

> 15.主键索引禁止删除.

> 16.禁止删除索引,请联系DBA确认.

## 其他规范

> 1.use db_name语句是多余的请删除.

> 2.请将关于一个表的操作合并到一条SQL中操作.

> 3.禁止跨库操作.

> 4.禁止insert into select ...的语法(大事物难以预估影响).

> 5.DML操作必须有where条件.

> 6.DML操作不能有limit条件.

> 7.禁止Truncate操作.

> 8.禁止Drop Table操作.

> 9.禁止Drop Database操作(高危操作).

> 10.禁止使用select *

> 11.禁止超过3个以上的表join，核心业务场景禁止join(可以适度冗余字段来避免jion，或者jion被下推到业务层处理).