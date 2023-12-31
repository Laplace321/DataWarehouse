# 行列转换介绍
## 背景说明
之所以写这篇文章是因为这个是问题是每个数据开发者从 0 开始学习时一定会遇到的问题，也是初级面试中高频出现和工作中经常遇到的问题。
然而根据我个人的经验，现在中文环境下的 hive 教程里，很难找到一个让我自己满意的行列转换教程。要么是对这个话题浅尝辄止，要么是太过深入于 explode 和 collect 函数而忽略了对本身行列转换应用场景的说明。基于此，本文将从行列转换的基本介绍，基础方法，进阶方法和我进一步的思考几个方面尽可能对行列转换这个话题进行详细和完整的介绍。
## 什么是行列转换
行列转换是数据开发者必备的技能，在实际的工作场景中经常需要对 hive 表中的数据进行行列转换操作。

所谓行列转换，就是将 hive 表中的一行多列数据转换成多行，或者将多行一列数据转换成多行，只改变数据的形式，不改变数据的内容。 



## 行转列
### 直接转换
最常见的行转列的方式就是将多行的数据转换成一列的字段显示，例如：

原始数据表
有一张成绩表 A（Name，Course，Score），主键是姓名和课程，每条记录统计该名学生某个课程的成绩：
|  Name   | Course  | Score |
|  ----  | ----  | ----|
| A  |  Math |  90 |
| A  |  Chinese |  91 |
| B  |  Math |  92 |
| B |  Chinese |  93 |

现在期望将这张表转换成以学生姓名为主键，每条记录统计了该名学生所有课程的成绩，即形式转换为：
|  Name   | Math  | Chinese |  
|  ----  | ----  | ----|
| A  |  90 |  91 |
| B |  92 |  93 |

用 sql 实现为：
```
select name
      ,max(case when course = 'Math' then Score else null end) as Math
      ,max(case when course = 'Chinese' then Score else null end) as Chinese
from A
group by name 

```
因为对于同一个学生，这里是把多行转成一行，所以需要对数据进行聚合。而转换的目标是当该列是特定课程的时候取出原表的值，同时需要满足聚合条件的使用规则。因此采用了 case when 和 max 的组合方法（这是一种进行行列转换时非常常用的方法）。

**这里插个题外话：如果你是有一定模型设计或优化经验的开发者，你应该能第一时间从上面这个例子中看出来，相较于原始表，转换后的新表在存储上具有更高的效率（存储节省了 50%），同时对于列存储的数据库来说，查询某个特定课程的分数计算效率也会提升。那这里是不是说新表的模型就一定优于旧表呢？我们考虑一个场景，这个课程体系增加新的课程或修改课程名称时，我们就不得不变更新表的表结构了，相比之下，旧表只需要新增或更新数据即可。我们知道，在数仓的模型设计中，数据模型的稳定性是一个很重要的指标，频繁的变更表结构不仅可能会带来一定的变更成本（尤其是对于需要补历史分区数据的情况），还会增加使用风险（变更期间可能会造成下游查询锁表、一些使用条件判断的查询出现问题等等）。因此虽然新表在存储和列查询效率上有所提升，但其可拓展性差对于模型稳定性的影响也不应被忽视，尤其是对于一些业务迭代变化较为频繁的场景，更应该考虑其频繁变更表结构的风险和成本。**
