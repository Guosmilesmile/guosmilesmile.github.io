---
title: Flink源码解析 SQL相关
date: 2019-12-14 17:05:38
tags:
categories:
	- Flink
---
![image](https://note.youdao.com/yws/api/personal/file/94A5670201B549D2993C0AD31FF26945?method=download&shareKey=29e08697de1d1482595102a287a7da5a)

一条stream sql从提交到calcite解析、优化最后到flink引擎执行，一般分为以下几个阶段:

```
1. Sql Parser: 将sql语句通过java cc解析成AST(语法树),在calcite中用SqlNode表示AST;
2. Sql Validator: 结合数字字典(catalog)去验证sql语法；
3. 生成Logical Plan: 将sqlNode表示的AST转换成LogicalPlan, 用relNode表示;
4. 生成 optimized LogicalPlan: 先基于calcite rules 去优化logical Plan,
再基于flink定制的一些优化rules去优化logical Plan；
5. 生成Flink PhysicalPlan: 这里也是基于flink里头的rules将，将optimized LogicalPlan转成成Flink的物理执行计划；
6. 将物理执行计划转成Flink ExecutionPlan: 就是调用相应的tanslateToPlan方法转换和利用CodeGen元编程成Flink的各种算子。
```

而如果是通过table api来提交任务的话，也会经过calcite优化等阶段，基本流程和直接运行sql类似:

```
1. table api parser: flink会把table api表达的计算逻辑也表示成一颗树，用treeNode去表式;
在这棵树上的每个节点的计算逻辑用Expression来表示。
2. Validate: 会结合数字字典(catalog)将树的每个节点的Unresolved Expression进行绑定，生成Resolved Expression；
3. 生成Logical Plan: 依次遍历数的每个节点，调用construct方法将原先用treeNode表达的节点转成成用calcite 内部的数据结构relNode 来表达。即生成了LogicalPlan, 用relNode表示;
4. 生成 optimized LogicalPlan: 先基于calcite rules 去优化logical Plan,
再基于flink定制的一些优化rules去优化logical Plan；
5. 生成Flink PhysicalPlan: 这里也是基于flink里头的rules将，将optimized LogicalPlan转成成Flink的物理执行计划；
6. 将物理执行计划转成Flink ExecutionPlan: 就是调用相应的tanslateToPlan方法转换和利用CodeGen元编程成Flink的各种算子。
```

```java

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);


		DataStream<Tuple2<String, Integer>> tuple2DataStreamSource = env.fromElements(Tuple2.of("guoyo1", 1), Tuple2.of("baiyx", 1));
		Table in = tableEnv.fromDataStream(tuple2DataStreamSource, "a,b");
		tableEnv.registerTable("MyTable", in);

		String sqlQuery = "SELECT * FROM MyTable";
		Table result = tableEnv.sqlQuery(sqlQuery);

		tableEnv.toAppendStream(result, Row.class).print();


		env.execute();
```
一行行代码来分析



```
Table in = tableEnv.fromDataStream(tuple2DataStreamSource, "a,b");
```
将dataStream转变为table的过程
```scala
@Override
	public <T> Table fromDataStream(DataStream<T> dataStream, String fields) {
		List<Expression> expressions = ExpressionParser.parseExpressionList(fields);
		JavaDataStreamQueryOperation<T> queryOperation = asQueryOperation(
			dataStream,
			Optional.of(expressions));

		return createTable(queryOperation);
	}
 ```
 ```
 ExpressionParser.parseExpressionList(fields);
```
将输入的字段名称，提取出来。


asQueryOperation返回的是JavaDataStreamQueryOperation，这个类用来描述DataSteam，对应的数据的索引和Tableschema（描述数据的字段名称、字段位置、字段类型）

asQueryOperation内进行了如下操作
* 将字段和index还有数据类型对应，得到 a,0,String。  b,1,Int
* 校验是否有是eventTime的流，如果是需要开启eventTime的相关配置
* 返回JavaDataStreamQueryOperation

然后获取到带有schema的DataStream，用于创建table  。
```java
createTable(queryOperation)
```

进去到里头，到了这层,将带有schema的DataStream转为了Table。先不细究里头的成员变量是什么用的。
```java
protected TableImpl createTable(QueryOperation tableOperation) {
		return TableImpl.createTable(
			this,
			tableOperation,
			operationTreeBuilder,
			functionCatalog);
	}
	
public static TableImpl createTable(
			TableEnvironment tableEnvironment,
			QueryOperation operationTree,
			OperationTreeBuilder operationTreeBuilder,
			FunctionLookup functionLookup) {
		return new TableImpl(
			tableEnvironment,
			operationTree,
			operationTreeBuilder,
			new LookupCallResolver(functionLookup));
	}
```


回到我们的代码
```java
tableEnv.registerTable("MyTable", in);
```
其中重要的部分是两个

```java
CatalogBaseTable tableTable = new QueryOperationCatalogView(table.getQueryOperation());
registerTableInternal(name, tableTable);
```
CatalogBaseTable是一个table的视图，以一个map维护table的键值对属性

在registerTableInternam中的主要注册table的方法是
```java
catalog.get().createTable(
					path,
					table,
					false);
```
这个Catalog是注册完成后数据库与数据表的原信息则存储在CataLog中。CataLog中保存了所有的表结构信息、数据目录信息等。

Catalog.createTable有两个具体的实现，一个是hive，是memory。
GenericInMemoryCatalog 将所有元数据存储在内存中，而 HiveCatalog 则通过 HiveShim 连接 Hive Metastore 的实例，提供元数据持久化的能力。通过 HiveCatalog，可以访问到 Hive 中管理的所有表，从而在 Batch 模式下使用。另外，通过 HiveCatalog 也可以使用 Hive 中的定义的 UDF，Flink SQL 提供了对于 Hive UDF 的支持。


至此，就把数据注册到catalog中。接下来就是对sql的解析了


```java
String sqlQuery = "SELECT a,sum(b) FROM MyTable group by a";
Table result = tableEnv.sqlQuery(sqlQuery);

```




TableEnvironmentImpl.sqlQuery中List<Operation> operations = planner.parse(query);开始了对sql的转换。

```scala

override def parse(stmt: String): util.List[Operation] = {
    val planner = createFlinkPlanner
    // parse the sql query  这一步解析得到 SqlNode
    val parsed = planner.parse(stmt)
    // SqlToOperationConverter 将 SqlNode 转化为 Operation
    parsed match {
      case insert: RichSqlInsert =>
        List(SqlToOperationConverter.convert(planner, insert))
      case query if query.getKind.belongsTo(SqlKind.QUERY) =>
        List(SqlToOperationConverter.convert(planner, query)) //查询语句
      case ddl if ddl.getKind.belongsTo(SqlKind.DDL) =>
        List(SqlToOperationConverter.convert(planner, ddl))
      case _ =>
        throw new TableException(s"Unsupported query: $stmt")
    }
  }
  
```
SQL 的解析在 PlannerBase.parse() 中实现：
1. 首先使用 Calcite 的解析出抽象语法树 SqlNode 
2. 然后结合元数据对 SQL 语句进行验证
3. 将 SqlNode 转换为 RelNode
4. 并最终封装为 Flink 内部对查询操作的抽象 QueryOperation。

```java
public static Operation convert(FlinkPlannerImpl flinkPlanner, SqlNode sqlNode) {
		// validate the query    结合元数据验证 Sql 的合法性
		final SqlNode validated = flinkPlanner.validate(sqlNode);
		// 将 SqlNode 转化为 Operation
		SqlToOperationConverter converter = new SqlToOperationConverter(flinkPlanner);
		if (validated instanceof SqlCreateTable) {
			return converter.convertCreateTable((SqlCreateTable) validated);
		} if (validated instanceof SqlDropTable) {
			return converter.convertDropTable((SqlDropTable) validated);
		} else if (validated instanceof RichSqlInsert) {
			return converter.convertSqlInsert((RichSqlInsert) validated);
		} else if (validated.getKind().belongsTo(SqlKind.QUERY)) {
			return converter.convertSqlQuery(validated);
		} else {
			throw new TableException("Unsupported node type "
				+ validated.getClass().getSimpleName());
		}
	}
	
	
	/** Fallback method for sql query. */
	private Operation convertSqlQuery(SqlNode node) {
		return toQueryOperation(flinkPlanner, node);
	}
```
Flink 借助于 Calcite 完成 SQl 的解析和优化，而后续的优化部分其实都是直接基于 RelNode 来完成的，那么这里为什么又多出了一个 QueryOperation 的概念呢？这主要是因为，Flink SQL 是支持 SQL 语句和 Table Api 接口混合使用的，在 Table Api 接口中，主要的操作都是基于 Operation 接口来完成的。

在校验这块，使用的是FlinkCalciteSqlValidator，继承了calcite的接口SqlValidatorImpl。所以才可以跟自己的schema串在一起。

### 怎么将schema注册到calcite中

DatabaseCalciteSchema 这个类是关键，这个类主要是用于将flink的schema转为Calcite's schema。实现了Calcite's schema接口。

CatalogManagerCalciteSchema--->CatalogCalciteSchema--->DatabaseCalciteSchema


如何将flink schema与calcite sheme结合起来呢。主要是PlannerBase这个类,将flink的CatalogManagerCalciteSchema转为calcite的SimpleCalciteSchema类。CatalogManagerCalciteSchema中的变量CatalogManager中存有我们通过flink注册的表信息。

```scala

private val plannerContext: PlannerContext =
    new PlannerContext(
      config,
      functionCatalog,
      asRootSchema(new CatalogManagerCalciteSchema(catalogManager, isStreamingMode)),
      getTraitDefs.toList
    )


public static CalciteSchema asRootSchema(Schema root) {
		return new SimpleCalciteSchema(null, root, "");
	}
```




### SQL 转换及优化


在将 SQL 语句解析成 Operation 后，为了得到 Flink 运行时的具体操作算子，需要进一步将 ModifyOperation 转换为 Transformation。在 Blink 之前的 SQL Planner 中，都是基于 DataStream 或 DataSet API 完成运行时逻辑的构建；而 Blink 则使用更底层的 Transformation 算子。

注意，Planner.translate(List<ModifyOperation> modifyOperations) 方法接收的参数是 ModifyOperation，ModifyOperation 对应的是一个 DML 的操作，在将查询结果插入到一张结果表或者转换为 DataStream 时，就会得到 ModifyOperation。

转换的流程主要分为四个部分，即
1. 将 Operation 转换为 RelNode
2. 优化 RelNode
3. 转换成 ExecNode
4. 转换为底层的 Transformation 算子。



```scala
abstract class PlannerBase(
    executor: Executor,
    config: TableConfig,
    val functionCatalog: FunctionCatalog,
    catalogManager: CatalogManager,
    isStreamingMode: Boolean)
  extends Planner {
  
  override def translate(
      modifyOperations: util.List[ModifyOperation]): util.List[Transformation[_]] = {
    if (modifyOperations.isEmpty) {
      return List.empty[Transformation[_]]
    }
    mergeParameters()
    // 1）将 Operation 转换为 RelNode
    val relNodes = modifyOperations.map(translateToRel)
    // 2）优化 RelNode
    val optimizedRelNodes = optimize(relNodes)
    // 3）转换成 ExecNode
    val execNodes = translateToExecNodePlan(optimizedRelNodes)
    // 4）转换为底层的 Transformation 算子
    translateToPlan(execNodes)
  }
}
```
首先需要进行的操作是将 Operation 转换为 RelNode，这个转换操作借助 QueryOperationConverter 完成

```
LogicalSink#2
  LogicalAggregate#1
    LogicalTableScan#0
```

在得到 RelNode 后，就进入 Calcite 对 RelNode 的优化流程。例如谓词下推之类的操作就是在这边完成的。


**在 Blink 中有一点特殊的地方在于，由于多个 RelNode 构成的树可能存在共同的“子树”（例如将相同的查询结果输出到不同的结果表中，那么两个 LogicalSink 的子树就可能是共用的），Blink 使用了一种 CommonSubGraphBasedOptimizer 优化器，将拥有共同子树的 RelNode 看作一个 DAG 结构，并将 DAG 划分成 RelNodeBlock，然后在RelNodeBlock 的基础上进行优化工作。每一个 RelNodeBlock 可以看作一个 RelNode 树进行优化，这和正常的 Calcite 处理流程还是保持一致的**(转载的，有待考究)


CommonSubGraphBasedOptimizer有两个实现，流的StreamCommonSubGraphBasedOptimizer和批的BatchCommonSubGraphBasedOptimizer

```scala
abstract class CommonSubGraphBasedOptimizer extends Optimizer {

  override def optimize(roots: Seq[RelNode]): Seq[RelNode] = {
  
    //以RelNodeBlock为单位进行优化，在子类中实现，StreamCommonSubGraphBasedOptimizer，BatchCommonSubGraphBasedOptimizer
    val sinkBlocks = doOptimize(roots)
    //获得优化后的逻辑计划
    val optimizedPlan = sinkBlocks.map { block =>
      val plan = block.getOptimizedPlan
      require(plan != null)
      plan
    }
    //将 RelNodeBlock 使用的中间表展开
    expandIntermediateTableScan(optimizedPlan)
    
  }
  
}
```

Caclite 对逻辑计划的优化是一套基于规则的框架，用户可以通过添加规则进行扩展，Flink 就是基于自定义规则来实现整个的优化过程。Flink 构造了一个链式的优化程序，可以按顺序使用多套规则集合完成 RelNode 的优化过程。

在 FlinkStreamProgram 和 FlinkBatchProgram 中定义了一系列扩展规则，用于构造逻辑计划的优化器。与此同时，Flink 扩展了 RelNode，增加了 FlinkLogicRel 和 FlinkPhysicRel 这两类 RelNode，对应的 Convention 分别为 FlinkConventions.LOGICAL 和 FlinkConventions.STREAM_PHYSICAL (或FlinkConventions.BATCH_PHYSICAL)。在优化器的处理过程中，RelNode 会从 Calcite 内部定义的节点转换为 FlinkLogicRel 节点（FlinkConventions.LOGICAL），并最终被转换为 FlinkPhysicRel 节点（FlinkConventions.STREAM_PHYSICAL）。这两类转换规则分别对应 FlinkStreamRuleSets.LOGICAL_OPT_RULES 和 FlinkStreamRuleSets.PHYSICAL_OPT_RULES。在不考虑其它更复杂的性能优化的情况下，如果要扩展 Flink SQL 的语法规则，可以参考这两类规则来增加节点和转换规则。


例如LogicSink在StreamCommonSubGraphBasedOptimizer.doOptimize会经过FlinkStreamProgram经过FlinkStreamRuleSets转为FlinkLogicalSink在转为StreamExecSinkRule。

![image](https://note.youdao.com/yws/api/personal/file/E5D9E234D91F42E2B8B8881315B2FD45?method=download&shareKey=ef4b3e8f4bdefd274b2686f0a03178a9)

经过优化器处理后，得到的逻辑树中的所有节点都应该是 FlinkPhysicRel，这之后就可以用于生成物理执行计划了。首先要将 FlinkPhysicalRel 构成的 DAG 转换成 ExecNode 构成的 DAG，因为可能存在共用子树的情况，这里还会尝试共用相同的子逻辑计划。由于通常 FlinkPhysicalRel 的具体实现类通常也实现了 ExecNode 接口，所以这一步转换较为简单。

在得到由 ExecNode 构成的 DAG 后，就可以尝试生成物理执行计划了，也就是将 ExecNode 节点转换为 Flink 内部的 Transformation 算子。不同的 ExecNode 按照各自的需求生成不同的 Transformation，基于这些 Transformation 构建 Flink 的 DAG。


### SQL 执行








### calcite相关


![image](https://note.youdao.com/yws/api/personal/file/EEC39CE3D2894B62BA7F614D0279A9E5?method=download&shareKey=41e918eb95a7d9dbd3ba14d19a3336d7)


* 关系代数（Relational algebra）：即关系表达式。它们通常以动词命名，例如 Sort, Join, Project, Filter, Scan, Sample.
* 行表达式（Row expressions）：例如 RexLiteral (常量), RexVariable (变量), RexCall (调用) 等，例如投影列表（Project）、过滤规则列表（Filter）、JOIN 条件列表和 ORDER BY 列表、WINDOW 表达式、函数调用等。使用 RexBuilder 来构建行表达式。
* 表达式有各种特征（Trait）：使用 Trait 的 satisfies() 方法来测试某个表达式是否符合某 Trait 或 Convention.
转化特征（Convention）：属于 Trait 的子类，用于转化 RelNode 到具体平台实现（可以将下文提到的 Planner 注册到 Convention 中）. 例如 JdbcConvention，FlinkConventions.DATASTREAM 等。同一个关系表达式的输入必须来自单个数据源，各表达式之间通过 Converter 生成的 Bridge 来连接。
* 规则（Rules）：用于将一个表达式转换（Transform）为另一个表达式。它有一个由 RelOptRuleOperand 组成的列表来决定是否可将规则应用于树的某部分。
* 规划器（Planner） ：即请求优化器，它可以根据一系列规则和成本模型（例如基于成本的优化模型 VolcanoPlanner、启发式优化模型 HepPlanner）来将一个表达式转为语义等价（但效率更优）的另一个表达式。




### Reference

[Flink 源码阅读笔记（15）- Flink SQL 整体执行框架](https://blog.jrwang.me/2019/flink-source-code-sql-overview/)


https://blog.csdn.net/qq475781638/article/details/92631194

https://www.infoq.cn/article/flink-api-table-api-and-sql

https://www.cnblogs.com/WCFGROUP/p/9241884.html

http://wuchong.me/blog/2017/03/30/flink-internals-table-and-sql-api/

https://www.cnblogs.com/029zz010buct/p/10142264.html

https://www.jianshu.com/p/6ed368272916

[Apache Calcite 功能简析及在 Flink 的应用](https://cloud.tencent.com/developer/article/1243475?fromSource=waitui)

[Apache Calcite 处理流程详解](https://matt33.com/2019/03/07/apache-calcite-process-flow)


