+++
title = "MySQL RC 隔离级别下的 GAP 死锁分析"
date = 2025-08-09
draft = false
tags = ["技术", "MySQL"]
summary = "RC 隔离级别下通常认为 Gap Lock 会被关闭，为什么仍然出现 GAP 死锁？通过简化模型复现 DEL + INSERT 死锁，分析唯一键检查的 Next-Key Lock 成因与三种解法。"
+++

## 背景

在一次错误日志收敛专项中，发现 MySQL 5.7、RC 隔离级别下出现了 GAP 死锁。

业务场景大致是：

- 订单完成后，将支付方式的支付金额均摊到订单项。
- 表上存在唯一键：`order_no + order_item_no`。
- 事务内操作模式为：先 `DELETE`，再 `INSERT`。

从死锁日志看，出现了 GAP / Next-Key Lock 相关的死锁。RC 隔离级别下通常认为 Gap Lock 会被关闭，但为什么这里仍然出现 GAP 死锁？本文通过简化模型复现并分析原因。

---

## 简化复现

假设表结构如下：

```sql
CREATE TABLE t_order_item (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  order_no VARCHAR(32) NOT NULL,
  order_item_no VARCHAR(32) NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  UNIQUE KEY uk_order_item (order_no, order_item_no)
) ENGINE=InnoDB;
```

假设表中已存在记录 `x`：

```sql
x = (order_no = 'O1', order_item_no = 'I1')
```

两个事务并发执行：

| 时序 | S1 事务 | S2 事务 | 说明 |
| --- | --- | --- | --- |
| T1 | `DELETE x` |  | S1 持有 `x` 的 X record lock |
| T2 |  | `DELETE x` | S2 等待 S1 释放 X record lock |
| T3 | `INSERT x` |  | S1 做唯一键冲突检查，请求 S Next-Key Lock，进入等待 |

最终形成死锁：

- S2 等待 S1 释放记录锁；
- S1 在 `INSERT` 时做唯一键冲突检查，需要请求 S Next-Key Lock；
- 该请求与索引上的等待队列冲突，S1 又开始等待 S2；
- 资源互持、相互等待，死锁形成。

---

## 死锁成因分析

核心矛盾在于：

1. S1 先删除记录 `x`，持有该记录的 X 锁。
2. S2 也删除记录 `x`，但被 S1 的 X 锁阻塞，进入等待队列。
3. S1 随后插入同一条唯一键记录 `x`，需要做唯一键冲突检查。
4. 唯一键冲突检查会请求 S Next-Key Lock。
5. 此时索引记录上已经存在等待队列，S1 的 S 锁请求与 S2 的等待请求形成循环依赖。

简化成等待关系：

```text
S2 等待 S1 释放 X 锁
S1 等待 S2 前面的等待请求 / 索引等待队列
```

于是形成死锁。

---

## 为什么唯一键检查必须用 S Next-Key Lock？

在 RC 隔离级别下，普通查询和索引扫描通常会关闭 Gap Lock，但唯一键冲突检查、外键检查等场景仍然可能使用 Next-Key Lock。

原因主要有：

1. **索引页上可能存在多条索引记录**
    - 唯一键检查并不是简单地对某一条记录加锁。
    - 为了保证检查区间稳定，需要使用 Next-Key Lock 防止其他事务在检查期间插入。

2. **索引可能跨页存在**
    - 唯一索引的检查范围可能跨越多个索引页。
    - 如果只加记录锁，无法阻止检查期间其他事务插入新的唯一键记录。

3. **二级唯一索引不包含完整行版本信息**
    - 主键聚簇索引叶子节点自带 `TRX_ID`，可以判断记录版本。
    - 二级唯一索引没有完整的 `TRX_ID`，需要回表查询。
    - 为了防止回表期间被其他事务插队，必须使用间隙锁 / Next-Key Lock 保护检查区间。

因此，在 `INSERT` 触发唯一键冲突检查时，InnoDB 可能加 S Next-Key Lock，而不是简单的记录锁。

---

## 主键会出现这个问题吗？

通常不会。

主键索引和二级唯一索引的结构不同：

- 主键是聚簇索引，叶子节点直接存储行数据，包含 `TRX_ID`、`ROLL_PTR` 等版本信息。
- 主键唯一性检查可以直接通过记录锁加版本可见性判断完成。
- 不需要额外通过 Gap Lock / Next-Key Lock 防止回表期间被插队。

而二级唯一索引不包含完整行数据，必须回表确认，因此更容易出现 Next-Key Lock 相关的等待和死锁。

---

## 解决方案

### 方案一：先 INSERT，再 DELETE

将事务内 SQL 顺序从：

```sql
DELETE FROM t_order_item WHERE order_no = ? AND order_item_no = ?;
INSERT INTO t_order_item (order_no, order_item_no, amount) VALUES (?, ?, ?);
```

调整为：

```sql
INSERT INTO t_order_item (order_no, order_item_no, amount) VALUES (?, ?, ?);
DELETE FROM t_order_item WHERE order_no = ? AND order_item_no = ?;
```

这样可以避免「删除后立刻插入同一唯一键」时，唯一键冲突检查的 S Next-Key Lock 与其他事务的删除等待形成环。

但需要注意：

- 如果新旧记录的唯一键完全相同，先 `INSERT` 会直接唯一键冲突。
- 这种情况下不应使用 `DEL + INSERT`，而应改为 `UPDATE` 或 `INSERT ... ON DUPLICATE KEY UPDATE`。

---

### 方案二：避免 DEL + INSERT，改为 UPSERT

更推荐的做法是使用 `INSERT ... ON DUPLICATE KEY UPDATE`：

```sql
INSERT INTO t_order_item (order_no, order_item_no, amount)
VALUES (?, ?, ?)
ON DUPLICATE KEY UPDATE amount = VALUES(amount);
```

或者先 `UPDATE`，影响行数为 0 时再 `INSERT`：

```sql
UPDATE t_order_item
SET amount = ?
WHERE order_no = ? AND order_item_no = ?;

-- 如果 affected rows = 0
INSERT INTO t_order_item (order_no, order_item_no, amount)
VALUES (?, ?, ?);
```

这样可以避免同一事务内对同一唯一键先删后插，从根源上减少死锁概率。

---

### 方案三：统一加锁顺序，缩短事务

如果业务必须进行删除和插入，可以考虑：

- 按 `order_no` 维度串行化，例如使用分布式锁或 `SELECT ... FOR UPDATE`。
- 统一多个事务的加锁顺序，避免交叉等待。
- 缩短事务执行时间，减少锁持有时间。
- 对死锁异常做重试处理。

---

## 总结

- RC 隔离级别不等于完全没有 Gap Lock。
- 唯一键冲突检查、外键检查等场景仍可能使用 Next-Key Lock。
- `DEL + INSERT` 同一唯一键在并发下容易形成「记录锁 + 唯一键检查 S Next-Key Lock」的循环等待。
- 主键聚簇索引自带 `TRX_ID`，记录锁通常足够；二级唯一索引需要回表，因此更容易触发 Next-Key Lock。
- 优先使用 `UPDATE` / `UPSERT` 替代 `DEL + INSERT`。
- 如果必须删插，可调整为「先插入，再删除」，并确保唯一键不冲突。
- 根因不是 GAP 锁本身，而是事务内锁顺序与唯一键检查锁冲突共同作用的结果。
