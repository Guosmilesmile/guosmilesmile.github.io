---
title: Calcite RBO rule 解析和自定义
date: 2020-08-11 10:11:05
tags:
categories: Calcite
---




### 什么是查询优化器

查询优化器是传统数据库的核心模块，也是大数据计算引擎的核心模块，开源大数据引擎如 Impala、Presto、Drill、HAWQ、 Spark、Hive 等都有自己的查询优化器。Calcite 就是从 Hive 的优化器演化而来的。

优化器的作用：将解析器生成的关系代数表达式转换成执行计划，供执行引擎执行，在这个过程中，会应用一些规则优化，以帮助生成更高效的执行计划。


### 基于成本优化（CBO）

基于代价的优化器(Cost-Based Optimizer，CBO)：根据优化规则对关系表达式进行转换，这里的转换是说一个关系表达式经过优化规则后会生成另外一个关系表达式，同时原有表达式也会保留，经过一系列转换后会生成多个执行计划，然后 CBO 会根据统计信息和代价模型 (Cost Model) 计算每个执行计划的 Cost，从中挑选 Cost 最小的执行计划。

由上可知，CBO 中有两个依赖：统计信息和代价模型。统计信息的准确与否、代价模型的合理与否都会影响 CBO 选择最优计划。 从上述描述可知，CBO 是优于 RBO 的，原因是 RBO 是一种只认规则，对数据不敏感的呆板的优化器，而在实际过程中，数据往往是有变化的，通过 RBO 生成的执行计划很有可能不是最优的。事实上目前各大数据库和大数据计算引擎都倾向于使用 CBO，但是对于流式计算引擎来说，使用 CBO 还是有很大难度的，因为并不能提前预知数据量等信息，这会极大地影响优化效果，CBO 主要还是应用在离线的场景。


### 基于规则优化（RBO）

基于规则的优化器（Rule-Based Optimizer，RBO）：根据优化规则对关系表达式进行转换，这里的转换是说一个关系表达式经过优化规则后会变成另外一个关系表达式，同时原有表达式会被裁剪掉，经过一系列转换后生成最终的执行计划。

RBO 中包含了一套有着严格顺序的优化规则，同样一条 SQL，无论读取的表中数据是怎么样的，最后生成的执行计划都是一样的。同时，在 RBO 中 SQL 写法的不同很有可能影响最终的执行计划，从而影响执行计划的性能。

### 优化规则


无论是 RBO，还是 CBO 都包含了一系列优化规则，这些优化规则可以对关系表达式进行等价转换，常见的优化规则包含：


1. 谓词下推 Predicate Pushdown
2. 常量折叠 Constant Folding
3. 列裁剪
4. 其他

在 Calcite 的代码里，有一个测试类（org.apache.calcite.test.RelOptRulesTest）汇集了对目前内置所有 Rules 的测试 case，这个测试类可以方便我们了解各个 Rule 的作用。

RBO的规则在Calite 1.24版本的时候，集中在org.apache.calcite.rel.rules.CoreRules中

在这里有下面一条 SQL，通过这条语句来说明一下上面介绍的这些种规则。

```
select 10 + 30, users.name, users.age
from users join jobs on users.id= user.id
where users.age > 30 and jobs.id>10

```
### 谓词下推（Predicate Pushdown）

关于谓词下推，它主要还是从关系型数据库借鉴而来，关系型数据中将谓词下推到外部数据库用以减少数据传输；属于逻辑优化，优化器将谓词过滤下推到数据源，使物理执行跳过无关数据。最常见的例子就是 join 与 filter 操作一起出现时，提前执行 filter 操作以减少处理的数据量，将 filter 操作下推，以上面例子为例，示意图如下

（对应 Calcite 中的CoreRules.FILTER_INTO_JOIN):

![image](https://note.youdao.com/yws/api/personal/file/F9E42F67D82149FEA470F837FABD8530?method=download&shareKey=3db6ce4e929c9b1efc0f836f4cac021c)


在进行 join 前进行相应的过滤操作，可以极大地减少参加 join 的数据量。


### 常量折叠（Constant Folding）

常量折叠也是常见的优化策略，这个比较简单、也很好理解，可以看下 编译器优化 – 常量折叠 这篇文章，基本不用动脑筋就能理解，对于我们这里的示例，有一个常量表达式 10 + 30，如果不进行常量折叠，那么每行数据都需要进行计算，进行常量折叠后的结果如下图所示

（ 对应 Calcite 中的 中的CoreRules.PROJECT_REDUCE_EXPRESSIONS Rule）：

![image](https://note.youdao.com/yws/api/personal/file/3263AA99C4BD46C9843476646FCE3A4D?method=download&shareKey=c90b340b2ad6f9f3ec859f2e784ee1db)

###　列裁剪（Column Pruning）

列裁剪也是一个经典的优化规则，在本示例中对于jobs 表来说，并不需要扫描它的所有列值，而只需要列值 id，所以在扫描 jobs 之后需要将其他列进行裁剪，只留下列 id。这个优化带来的好处很明显，大幅度减少了网络 IO、内存数据量的消耗。裁剪前后的示意图如下（不过并没有找到 Calcite 对应的 Rule）：

![image](https://note.youdao.com/yws/api/personal/file/7F01E2D51AC34EF5B32CD131C5DA19FD?method=download&shareKey=4a133510b85532b7c7ab15d9a2085335)


### 如何使用RBO


```java
public class CalciteRuleTest {


    public static void main(String[] args) throws SqlParseException, ValidationException, RelConversionException {

        Planner planner = getPlanner();

        SqlNode sqlNode = planner.parse("SELECT DATE_CD,SUM(IB0002001_CN000) FROM \n" +
                "(SELECT IDX_ID,CUBE2L_IB00040010_CN000.DATE_CD,SUM(CUBE2L_IB00040010_CN000.IDX_VAL) AS IB0002001_CN000 FROM " +
                "CUBE2L_IB00040010_CN000" +
                " GROUP BY  CUBE2L_IB00040010_CN000.DATE_CD,IDX_ID) IB0002001_CN000  WHERE IDX_ID IN ('IB0002001_CN000') AND DATE_CD = '2020-05-31' GROUP BY DATE_CD ");



        System.out.println(sqlNode.toString());

        planner.validate(sqlNode);
        RelRoot relRoot = planner.rel(sqlNode);

        RelNode relNode = relRoot.project();

        System.out.println();
        System.out.print(RelOptUtil.toString(relNode));


        HepProgramBuilder builder = new HepProgramBuilder();
//        builder.addRuleInstance(CoreRules.FILTER_MERGE); //note: 添加 rule
//        builder.addRuleInstance(CoreRules.PROJECT_FILTER_TRANSPOSE);
//        builder.addRuleInstance(CoreRules.FILTER_AGGREGATE_TRANSPOSE);
//        builder.addRuleInstance(CoreRules.FILTER_PROJECT_TRANSPOSE);
//        builder.addRuleInstance(CoreRules.FILTER_AGGREGATE_TRANSPOSE);
        builder.addRuleInstance(CoreRules.AGGREGATE_REMOVE);
        builder.addRuleInstance(CoreRules.PROJECT_FILTER_TRANSPOSE);
        HepPlanner hepPlanner = new HepPlanner(builder.build());
        hepPlanner.setRoot(relNode);
        relNode = hepPlanner.findBestExp();

        System.out.println();
        System.out.println("After --------------------");
        System.out.println();
        System.out.print(RelOptUtil.toString(relNode));


        RelToSqlConverter relToSqlConverter = new RelToSqlConverter(MysqlSqlDialect.DEFAULT);
        SqlImplementor.Result result = relToSqlConverter.visitRoot(relNode);


        System.out.println();
        System.out.println(result.asSelect().toString());
    }


    private static Planner getPlanner() {
        SchemaPlus rootSchema = Frameworks.createRootSchema(true);


        // 添加表的schema信息，才可以通过validate
        rootSchema.add("CUBE2L_IB00040010_CN000", new AbstractTable() {
            public RelDataType getRowType(final RelDataTypeFactory typeFactory) {

                RelDataTypeFactory.Builder builder = typeFactory.builder();
                builder.add("DATE_CD", typeFactory.createTypeWithNullability(typeFactory.createSqlType(SqlTypeName.DATE), true));
                builder.add("IDX_VAL", typeFactory.createTypeWithNullability(typeFactory.createSqlType(SqlTypeName.BIGINT), true));
                builder.add("test", typeFactory.createTypeWithNullability(typeFactory.createSqlType(SqlTypeName.BIGINT), true));
                builder.add("IDX_ID", typeFactory.createTypeWithNullability(typeFactory.createSqlType(SqlTypeName.VARCHAR), true));

                return builder.build();
            }
        });

        SqlParser.ConfigBuilder parserConfig = SqlParser.configBuilder();
        SqlParser.Config build = parserConfig.setCaseSensitive(false).setLex(Lex.MYSQL).build();

        FrameworkConfig frameworkConfig = Frameworks.newConfigBuilder()
                .parserConfig(build)
                .defaultSchema(rootSchema)
                .build();

        Planner planner = Frameworks.getPlanner(frameworkConfig);
        return planner;
    }

```

上面的代码总共分为三步：

1. 初始化 HepProgram 对象；
2. 初始化 HepPlanner 对象，并通过 setRoot() 方法将 RelNode 树转换成 HepPlanner 内部使用的 Graph；
3. 通过 findBestExp() 找到最优的 plan，规则的匹配都是在这里进行。


#### 1. 初始化 HepProgram

这几步代码实现没有太多需要介绍的地方，先初始化 HepProgramBuilder 也是为了后面初始化 HepProgram 做准备，HepProgramBuilder 主要也就是提供了一些配置设置和添加规则的方法等，常用的方法如下：

1. addRuleInstance()：注册相应的规则；
2. addRuleCollection()：这里是注册一个规则集合，先把规则放在一个集合里，再注册整个集合，如果规则多的话，一般是这种方式；
3. addMatchLimit()：设置 MatchLimit，这个 rule match 次数的最大限制；
4. addMatchOrder() Rule 匹配的顺序


HepProgram 这个类对于后面 HepPlanner 的优化很重要，它定义 Rule 匹配的顺序，默认按【深度优先】顺序，它可以提供以下几种（见 HepMatchOrder 类）：

1. ARBITRARY：按任意顺序匹配（因为它是有效的，而且大部分的 Rule 并不关心匹配顺序）；
2. BOTTOM_UP：自下而上，先从子节点开始匹配；
3. TOP_DOWN：自上而下，先从父节点开始匹配；
4. DEPTH_FIRST：深度优先匹配，某些情况下比 ARBITRARY 高效（为了避免新的 vertex 产生后又从 root 节点开始匹配）。


这个匹配顺序到底是什么呢？对于规则集合 rules，HepPlanner 的算法是：从一个节点开始，跟 rules 的所有 Rule 进行匹配，匹配上就进行转换操作，这个节点操作完，再进行下一个节点，这里的匹配顺序就是指的节点遍历顺序（这种方式的优劣，我们下面再说）。




### 分析现有rule

最简单的例子，Join结合律，JoinAssociateRule



首先所有的Rule都继承RelOptRule类

```java
/**
 * Planner rule that changes a join based on the associativity rule.
 *
 * <p>((a JOIN b) JOIN c) &rarr; (a JOIN (b JOIN c))</p>
 *
 * <p>We do not need a rule to convert (a JOIN (b JOIN c)) &rarr;
 * ((a JOIN b) JOIN c) because we have
 * {@link JoinCommuteRule}.
 *
 * @see JoinCommuteRule
 */
public class JoinAssociateRule extends RelOptRule implements TransformationRule 

```

对于Join结合律，调用super，即RelOptRule的构造函数,匹配的树状为
```
join
   join 
      any
```
```java
 /**
   * Creates a JoinAssociateRule.
   */
  public JoinAssociateRule(RelBuilderFactory relBuilderFactory) {
    super(
        operand(Join.class,
            operand(Join.class, any()),
            operand(RelSubset.class, any())),
        relBuilderFactory, null);
  }
```
operand也是一个树形结构，Top的Operand的类型是Join，他有两个children，其中一个也是join，另一个是RelSubset

他们的children是any()


```java
public void onMatch(final RelOptRuleCall call) {
    final Join topJoin = call.rel(0);
    final Join bottomJoin = call.rel(1);
    final RelNode relA = bottomJoin.getLeft();
    final RelNode relB = bottomJoin.getRight();
    final RelSubset relC = call.rel(2);
    final RelOptCluster cluster = topJoin.getCluster();
    final RexBuilder rexBuilder = cluster.getRexBuilder();

    if (relC.getConvention() != relA.getConvention()) {
      // relC could have any trait-set. But if we're matching say
      // EnumerableConvention, we're only interested in enumerable subsets.
      return;
    }

    //        topJoin
    //        /     \
    //   bottomJoin  C
    //    /    \
    //   A      B

    final int aCount = relA.getRowType().getFieldCount();
    final int bCount = relB.getRowType().getFieldCount();
    final int cCount = relC.getRowType().getFieldCount();
    final ImmutableBitSet aBitSet = ImmutableBitSet.range(0, aCount);
    final ImmutableBitSet bBitSet =
        ImmutableBitSet.range(aCount, aCount + bCount);

    if (!topJoin.getSystemFieldList().isEmpty()) {
      // FIXME Enable this rule for joins with system fields
      return;
    }

    // If either join is not inner, we cannot proceed.
    // (Is this too strict?)
    if (topJoin.getJoinType() != JoinRelType.INNER
        || bottomJoin.getJoinType() != JoinRelType.INNER) {
      return;
    }

    // Goal is to transform to
    //
    //       newTopJoin
    //        /     \
    //       A   newBottomJoin
    //               /    \
    //              B      C

    // Split the condition of topJoin and bottomJoin into a conjunctions. A
    // condition can be pushed down if it does not use columns from A.
    final List<RexNode> top = new ArrayList<>();
    final List<RexNode> bottom = new ArrayList<>();
    JoinPushThroughJoinRule.split(topJoin.getCondition(), aBitSet, top, bottom);
    JoinPushThroughJoinRule.split(bottomJoin.getCondition(), aBitSet, top,
        bottom);

    // Mapping for moving conditions from topJoin or bottomJoin to
    // newBottomJoin.
    // target: | B | C      |
    // source: | A       | B | C      |
    final Mappings.TargetMapping bottomMapping =
        Mappings.createShiftMapping(
            aCount + bCount + cCount,
            0, aCount, bCount,
            bCount, aCount + bCount, cCount);
    final List<RexNode> newBottomList =
        new RexPermuteInputsShuttle(bottomMapping, relB, relC)
            .visitList(bottom);
    RexNode newBottomCondition =
        RexUtil.composeConjunction(rexBuilder, newBottomList);

    final Join newBottomJoin =
        bottomJoin.copy(bottomJoin.getTraitSet(), newBottomCondition, relB,
            relC, JoinRelType.INNER, false);

    // Condition for newTopJoin consists of pieces from bottomJoin and topJoin.
    // Field ordinals do not need to be changed.
    RexNode newTopCondition = RexUtil.composeConjunction(rexBuilder, top);
    @SuppressWarnings("SuspiciousNameCombination")
    final Join newTopJoin =
        topJoin.copy(topJoin.getTraitSet(), newTopCondition, relA,
            newBottomJoin, JoinRelType.INNER, false);

    call.transformTo(newTopJoin);
  }
```

### 自定义rule

补充下基础概念

RexLiteral表示常量，RexVariable表示变量，RexCall表示操作来连接Literal和Variable



自定义rule，有三个主要的方法，一个是matches，一个是onMatch，一个是构造函数

判断一个tree是否满足该rule，先从构造函数开始匹配，满足构造函数的tree，在满足matches方法，就可以进入到onMatch方法。

matches方法默认是返回true，可以进行改写

```java
 @Override
    public boolean matches(RelOptRuleCall call) {
        return super.matches(call);
    }
```

假设想要匹配

```
join
  project
```
构造函数得这么写

```
 super(
        operand(Join.class,
            operand(Project.class, any())),
        relBuilderFactory, null);
```

在onMatch中可以通过call.rel(x)获取构造函数中匹配的relNode

```
Join join = call.rel(0);
Project project = call.rel(1);

```

通过一些强转可以获取对应的节点

```java
((HepRelVertex)rel.getInput(0)).getCurrentRel()
```

创建rebuild，生成node

```java
 RelBuilder relBuilder = call.builder();
 RelNode build = relBuilder.build();
```

对relBuilder操作

```java

relBuilder.push 加入数据源
relbuilder.project 加入投影 还可以agg filter 等等
```


通过call替换掉原来的tree

```java
call.transformTo(node);
```


### 常用函数

```
RexUtil.composeDisjunction 以or合并两个规则

RexUtil.composeConjunction 以and合并两个规则

((HepRelVertex)rel.getInput(0)).getCurrentRel() 强转获取节点

node.getCluster()可以继续get出很多东西getPlanner()、getRexBuilder()、getTypeFactory()

RelDataType sqlType = typeFactory.createSqlType(SqlTypeName.ANY) 创建对应的类型

rexBuilder.makeLiteral 创建常量  makeCall创建操作符等等

SqlStdOperatorTable可以获取函数集合  SUM MIN MAX

创建agg  relBuilder.aggregate(relBuilder.groupKey(ImmutableBitset.range(aggCount)),aggCallLists)

```
### 中文编码问题

Calcite中默认使用的编码是ISO-8895-1，如果使用中文，就会出现编码问题，需要修改编码方式。

通过在代码中增加解决

```
System.setProperty("saffron.default.charset",ConversionUtil.NATIVE_UTF16_CHARSET_NAME);
```


### Reference
[Calcite分析 - Rule](https://www.cnblogs.com/fxjwind/p/11279080.html)

[Apache Calcite 优化器详解（二）](https://matt33.com/2019/03/17/apache-calcite-planner/)