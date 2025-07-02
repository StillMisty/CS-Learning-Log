# 关于PostgreSQL的一些笔记

## 1. Joined Tables

### 1.1 Join Types

1. Cross Join: 两个表的笛卡尔积
2. Inner Join: 两个表的交集
3. Left Join: 以左表为基准，右表中没有的数据为NULL
4. Right Join: 以右表为基准，左表中没有的数据为NULL
5. Full Join: 两个表的并集，没有的数据为NULL
6. Natural Join: 两个表的交集，但是不显示重复的列

### 1.2 Join Conditions

1. ON: 指定连接条件
2. USING: 指定连接的列，但是列名必须相同,不显示重复的列

``` sql
USING (A, B)
等价于，但是USING不显示重复的列
ON table1.A = table2.A AND table1.B = table2.B
```

## 2. Select Lists

### 2.1 保留关键词

SQL 区分保留关键字和非保留关键字。

- 保留关键字是唯一真正的关键字；它们绝不允许用作标识符。非保留关键字仅在特定上下文中具有特殊含义，可在其他上下文中用作标识符。大多数非保留关键字实际上是 SQL 指定的内置表和函数的名称。
- 非保留关键字的概念本质上只是为了声明在某些上下文中某个词具有某些预定义的含义。

*USER* 是一个保留关键字，不能用作标识符，用作列名时需要用双引号括起来。

``` sql
SELECT a "USER", b + c AS sum FROM
```

在PostgreSQL解析器中，情况稍微复杂一些。有几种不同类型的标记，从永远不能用作标识符的标记到在解析器中绝对没有特殊地位但被视为普通标识符的标记。（后者通常是 SQL 指定的函数的情况。）

即使是保留关键字在PostgreSQL中也不是完全保留的，但可以用作列标签（例如，SELECT 55 AS CHECK即使CHECK是保留关键字）。

[附录 C.  SQL关键词](https://www.postgresql.org/docs/current/sql-keywords-appendix.html)

## 2.2 SELECT DISTINCT

``` sql
SELECT DISTINCT column1, column2 FROM table_name;
```

如果两行至少有一列值不同，则认为它们不同。在此比较中，空值被视为相等。

或者，任意表达式可以确定哪些行被视为不同的行：

``` sql
SELECT DISTINCT ON (column1) column1, column2 FROM table_name;
```

此时，只有column1不同的行被视为不同的行。

该DISTINCT ON子句不是 SQL 标准的一部分，有时由于其结果可能具有不确定性而被认为是不好的风格。如果明智地使用 中的GROUP BY和子查询FROM，可以避免这种构造，但它通常是最方便的替代方法。

## Combining Queries (UNION, INTERSECT, EXCEPT)

### 3.1 语法

``` sql
query1 UNION [ALL] query2
query1 INTERSECT [ALL] query2
query1 EXCEPT [ALL] query2
```

- UNION: 返回两个查询的结果集的并集，去除重复的行
- INTERSECT: 返回两个查询的结果集的交集，去除重复的行
- EXCEPT: 返回两个查询的结果集的差集，去除重复的行
- 包含ALL关键字时，不去除重复的行

### 3.2 注意事项

必要时，可以使用括号来明确指定操作的顺序。

``` sql
SELECT a FROM b UNION SELECT x FROM y LIMIT 10
等价于
(SELECT a FROM b UNION SELECT x FROM y) LIMIT 10
```

## 4. Sorting Rows (ORDER BY)

### 4.1 语法

``` sql
SELECT select_list
    FROM table_expression
    ORDER BY sort_expression1 [ASC | DESC] [NULLS { FIRST | LAST }]
             [, sort_expression2 [ASC | DESC] [NULLS { FIRST | LAST }] ...]
```

排序表达式可以是列名或表达式，也可以是列名或表达式的列表。

``` sql
SELECT a, b FROM table1 ORDER BY a + b, c;
```

默认情况下，所有列都按升序ASC排序。要指定降序排序，请使用DESC关键字。

### 4.2 NULLS FIRST | NULLS LAST

NULLS FIRST和选项NULLS LAST可用于确定在排序顺序中，空值出现在非空值之前还是之后。

默认情况下，空值排序为大于任何非空值；也就是说，NULLS FIRST是 order 的默认设置。

### 4.3 注意事项

查询生成输出表后（处理选择列表后），可以选择对其进行排序。如果未选择排序，则行将以未指定的顺序返回。

在这种情况下，实际顺序将取决于扫描和连接计划类型以及磁盘上的顺序，但不能依赖它。

只有明确选择排序步骤，才能保证特定的输出顺序。

## 5. LIMIT and OFFSET

### 5.1 语法

``` sql
SELECT select_list
    FROM table_expression
    [ ORDER BY ... ]
    [ LIMIT { count | ALL } ]
    [ OFFSET start ]
```

LIMIT子句用于限制返回的行数，LIMIT ALL用于返回所有行。

OFFSET子句用于跳过前N行。

### 5.2 注意事项

使用LIMIT和OFFSET子句时，应该明确指定ORDER BY子句，否则结果是不确定的。

子句跳过的行OFFSET仍需在服务器内部进行计算；因此，较大的子句OFFSET可能效率低下。

当数据库收到一个带有 OFFSET 的查询时，它会执行以下步骤：

1. 读取数据: 数据库引擎会按照 WHERE 子句（如果有）的过滤条件读取表中的数据。

2. 排序数据: 如果查询包含 ORDER BY 子句，数据库会对读取的数据进行排序。

3. 跳过行: 数据库会根据 OFFSET 子句的值跳过指定的行数。

4. 返回剩余行: 数据库返回剩余的行，直到达到 LIMIT 子句指定的限制（如果有）

即使你只需要最后几行，数据库也必须执行步骤 1、2 和 3。 这意味着数据库必须读取、排序（如果适用）并丢弃大量数据，即使这些数据最终不会被返回。 这会导致大量的 I/O 操作和 CPU 处理，从而降低查询性能。

对于大型数据集和大的偏移量，使用基于键的分页通常效率更高。 这涉及到使用 WHERE 子句和一个索引列来过滤数据。 例如，可以使用上一页结果的最后一个值作为下一页的起始点。

如果你的表有一个自增主键 id，并且你上一页的最后一个 id 是 1000，你可以使用以下查询获取下一页的 100 行。

``` sql
SELECT * FROM my_table WHERE id > 1000 ORDER BY id LIMIT 100;
```

这种方法避免了读取和丢弃大量不需要的行，从而显著提高了查询性能。

## 6. VALUES Lists

``` sql
VALUES ( expression [, ...] ) [, ...]
```

VALUES子句用于生成一个表，其中包含指定的值列表。

``` sql
VALUES (1, 'one'), (2, 'two'), (3, 'three');
等价于
SELECT 1 AS column1, 'one' AS column2
UNION ALL
SELECT 2, 'two'
UNION ALL
SELECT 3, 'three';
```

默认情况下，PostgreSQL会将名称column1、column2等分配给VALUES的列。SQL 标准并未指定列名，而且不同的数据库系统对此的处理方式也不同，因此通常最好使用表别名列表覆盖默认名称，如下所示：

``` sql
SELECT * FROM (VALUES (1, 'one'), (2, 'two'), (3, 'three')) AS t (num,letter); 

 num | letter 
-----+-------- 
   1 | one 
   2 | two 
   3 | three 
```

VALUES最常用作INSERT命令中的数据源，其次最常用作子查询。

## 7. WITH Queries (Common Table Expressions)

### 7.1 语法

WITH子句允许你在查询中定义一个临时表，然后在查询中引用它。

``` sql
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
), top_regions AS (
    SELECT region
    FROM regional_sales
    WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
)
SELECT region,
       product,
       SUM(quantity) AS product_units,
       SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

它仅显示销售量最高的地区每种产品的销售总额。

该WITH子句定义了两个名为 regional_sales 和 top_regions 的辅助语句，其中 regional_sales 的输出用于 top_regions 的查询，top_regions的输出用于主SELECT查询。

### 7.2 Recursive Queries

WITH RECURSIVE子句允许你执行递归查询。

``` sql
WITH RECURSIVE t(n) AS (
    VALUES (1)
  UNION ALL
    SELECT n+1 FROM t WHERE n < 100
)
SELECT sum(n) FROM t;
```

这个查询计算从1到100的和。
