## 配置文件    
    
    采用 yaml 格式。配置用户、node、分片规则等。
    
    Rule.SubTableIndexs 中的 SubTable 指的是每一个分片表

## 后端数据库存活状态监测

    对每个node，启动一个goroutine，每隔16秒对master和所有的slave去ping一次。

## 后端连接池

    * 连接池：DB.cacheConns
    * 对象池：DB.idleConns

## 主从节点及负载均衡

非事务的select请求默认是按RoundRobin加上权重发给每个Slave，用户可以在Select SQL中用hint主动要求发给master。
insert,update,delete,replace 请求或者事务中的select请求直接发给master。

## 事务支持

    如果当前语句执行在事务环境下，且执行计划确定的node数大于1，则报错。

## SQL预处理

    * SQL黑名单处理
    * 不需分片的SQL提前处理

## 生成查询计划
    ### 目的
    
    判断返回值落在哪个分片上，缩小SQL语句的执行范围，只在相关的分片所在的node上执行。

    * 每个node对应一个物理连接，每个分片对应一个表，每个node可以对应多个表，每个表对应一条SQL语句。
    * 如果没有分片，则SQL发给第一个node（通常是缺省node）
    * 如果没有where条件，则SQL发给所有node的所有分片表

## 改写SQL
    + 在select xxx 中添加所有group by中的列（用于后面合并各分片的结果）
    + 修改表名为分片表名（列名前缀处，from处，Join的左边）
    + 如果where in的条件在某个分片上范围能缩小，则改写之
    + 修改limit为offset+count

## 合并结果
    1. 处理结果集合并
    
     - 没有group by列的情况:
```
        if (select列中不存在函数表达式） {
            简单合并所有分片的结果
        } else {
            仅存在单行结果集
        }
```
     - 有group by列的情况:
```
        if (select列中没有函数） {
             后一个分片覆盖前一个分片的值即可
        } else {
            需要以添加的所有group by列的组合做为key，按照每个函数的类型对结果集进行合并
        }
```
     需要处理的函数为：count，sum，max，min，last_insert_id 
    
    2. 处理排序（如果需要）
    3. 处理limit（如果需要）