---
title: SqlNode toString源码解析以及SqlOrderBy无法根据方言自适应的bug
date: 2020-10-31 15:58:44
tags:
categories:
	- Calcite
---



主要方法来来自于SqlNode.toSqlString
```java
public SqlString toSqlString(UnaryOperator<SqlWriterConfig> transform) {
    final SqlWriterConfig config = transform.apply(SqlPrettyWriter.config());
    SqlPrettyWriter writer = new SqlPrettyWriter(config);
    unparse(writer, 0, 0);
    return writer.toSqlString();
  }
```

主要的方法来源于unparse方法，这个方法在sqlNode中的定义如下.

将此节点的SQL表示写到写入器,leftPrec和rightPrec参数为我们提供了足够的上下文来决定是否需要将表达式用括号括起来。
   

```java
/**
   * Writes a SQL representation of this node to a writer.
   * <p>The <code>leftPrec</code> and <code>rightPrec</code> parameters give
   * us enough context to decide whether we need to enclose the expression in
   * parentheses. For example, we need parentheses around "2 + 3" if preceded
   * by "5 *". This is because the precedence of the "*" operator is greater
   * than the precedence of the "+" operator.
   *
   * <p>The algorithm handles left- and right-associative operators by giving
   * them slightly different left- and right-precedence.
   *
   * <p>If {@link SqlWriter#isAlwaysUseParentheses()} is true, we use
   * parentheses even when they are not required by the precedence rules.
   *
   * <p>For the details of this algorithm, see {@link SqlCall#unparse}.
   *
   * @param writer    Target writer
   * @param leftPrec  The precedence of the {@link SqlNode} immediately
   *                  preceding this node in a depth-first scan of the parse
   *                  tree
   * @param rightPrec The precedence of the {@link SqlNode} immediately
   */
public abstract void unparse(
      SqlWriter writer,
      int leftPrec,
      int rightPrec);

```

具体实现取决于子类的node,最基础的实现是SqlCall.unparse。SqlSelect/SqlOrderBy都有自己的实现，得到不同的sql

```java
public void unparse(
      SqlWriter writer,
      int leftPrec,
      int rightPrec) {
    final SqlOperator operator = getOperator();
    final SqlDialect dialect = writer.getDialect();
    if (leftPrec > operator.getLeftPrec()
        || (operator.getRightPrec() <= rightPrec && (rightPrec != 0))
        || writer.isAlwaysUseParentheses() && isA(SqlKind.EXPRESSION)) {
      final SqlWriter.Frame frame = writer.startList("(", ")");
      dialect.unparseCall(writer, this, 0, 0);
      writer.endList(frame);
    } else {
      dialect.unparseCall(writer, this, leftPrec, rightPrec);
    }
  }

```

先判断是否需要添加括号，如果不需要就直接进行dialect.unparseCall

如果需要添加括号那么就是，先申明一个Frame，startList的开和闭是(),
然后，再进行dialect.unparseCall后得到的结果以Frame结尾

```java
 final SqlWriter.Frame frame = writer.startList("(", ")");
 dialect.unparseCall(writer, this, 0, 0);
 writer.endList(frame);
```


SqlDialect.unparseCall有自己的实现，基本的实现如下

```java
 public void unparseCall(SqlWriter writer, SqlCall call, int leftPrec,
      int rightPrec) {
    switch (call.getKind()) {
    case ROW:
      // Remove the ROW keyword if the dialect does not allow that.
      if (!getConformance().allowExplicitRowValueConstructor()) {
        // Fix the syntax when there is no parentheses after VALUES keyword.
        if (!writer.isAlwaysUseParentheses()) {
          writer.print(" ");
        }
        final SqlWriter.Frame frame = writer.isAlwaysUseParentheses()
                ? writer.startList(SqlWriter.FrameTypeEnum.FUN_CALL)
                : writer.startList(SqlWriter.FrameTypeEnum.FUN_CALL, "(", ")");
        for (SqlNode operand : call.getOperandList()) {
          writer.sep(",");
          operand.unparse(writer, leftPrec, rightPrec);
        }
        writer.endList(frame);
        break;
      }
      call.getOperator().unparse(writer, call, leftPrec, rightPrec);
      break;
    default:
      call.getOperator().unparse(writer, call, leftPrec, rightPrec);
    }
  }


```
下沉到call.getOperator().unparse

先看看如果这个sqlNode是sqlSelect，对应的operation是SqlSelectOperator

SqlSelectOperator.unparse如下：
```java
public void unparse(
      SqlWriter writer,
      SqlCall call,
      int leftPrec,
      int rightPrec) {
    SqlSelect select = (SqlSelect) call;
    final SqlWriter.Frame selectFrame =
        writer.startList(SqlWriter.FrameTypeEnum.SELECT);
    writer.sep("SELECT");

    if (select.hasHints()) {
      writer.sep("/*+");
      select.hints.unparse(writer, 0, 0);
      writer.print("*/");
      writer.newlineAndIndent();
    }

    for (int i = 0; i < select.keywordList.size(); i++) {
      final SqlNode keyword = select.keywordList.get(i);
      keyword.unparse(writer, 0, 0);
    }
    writer.topN(select.fetch, select.offset);
    final SqlNodeList selectClause =
        select.selectList != null
            ? select.selectList
            : SqlNodeList.of(SqlIdentifier.star(SqlParserPos.ZERO));
    writer.list(SqlWriter.FrameTypeEnum.SELECT_LIST, SqlWriter.COMMA,
        selectClause);

    if (select.from != null) {
      // Calcite SQL requires FROM but MySQL does not.
      writer.sep("FROM");

      // for FROM clause, use precedence just below join operator to make
      // sure that an un-joined nested select will be properly
      // parenthesized
      final SqlWriter.Frame fromFrame =
          writer.startList(SqlWriter.FrameTypeEnum.FROM_LIST);
      select.from.unparse(
          writer,
          SqlJoin.OPERATOR.getLeftPrec() - 1,
          SqlJoin.OPERATOR.getRightPrec() - 1);
      writer.endList(fromFrame);
    }

    if (select.where != null) {
      writer.sep("WHERE");

      if (!writer.isAlwaysUseParentheses()) {
        SqlNode node = select.where;

        // decide whether to split on ORs or ANDs
        SqlBinaryOperator whereSep = SqlStdOperatorTable.AND;
        if ((node instanceof SqlCall)
            && node.getKind() == SqlKind.OR) {
          whereSep = SqlStdOperatorTable.OR;
        }

        // unroll whereClause
        final List<SqlNode> list = new ArrayList<>(0);
        while (node.getKind() == whereSep.kind) {
          assert node instanceof SqlCall;
          final SqlCall call1 = (SqlCall) node;
          list.add(0, call1.operand(1));
          node = call1.operand(0);
        }
        list.add(0, node);

        // unparse in a WHERE_LIST frame
        writer.list(SqlWriter.FrameTypeEnum.WHERE_LIST, whereSep,
            new SqlNodeList(list, select.where.getParserPosition()));
      } else {
        select.where.unparse(writer, 0, 0);
      }
    }
    if (select.groupBy != null) {
      writer.sep("GROUP BY");
      final SqlNodeList groupBy =
          select.groupBy.size() == 0 ? SqlNodeList.SINGLETON_EMPTY
              : select.groupBy;
      writer.list(SqlWriter.FrameTypeEnum.GROUP_BY_LIST, SqlWriter.COMMA,
          groupBy);
    }
    if (select.having != null) {
      writer.sep("HAVING");
      select.having.unparse(writer, 0, 0);
    }
    if (select.windowDecls.size() > 0) {
      writer.sep("WINDOW");
      writer.list(SqlWriter.FrameTypeEnum.WINDOW_DECL_LIST, SqlWriter.COMMA,
          select.windowDecls);
    }
    if (select.orderBy != null && select.orderBy.size() > 0) {
      writer.sep("ORDER BY");
      writer.list(SqlWriter.FrameTypeEnum.ORDER_BY_LIST, SqlWriter.COMMA,
          select.orderBy);
    }
    writer.fetchOffset(select.fetch, select.offset);
    writer.endList(selectFrame);
  }
 ```
 
### 坑

虽然sqlSelectOperation的unparse中通过writer来转为sql

```java
writer.fetchOffset(select.fetch, select.offset);
writer.endList(selectFrame);
```

但是在sqlorder中unparse却没有把offset和fetch传递到里头去，而是自己处理，这样就无法做到根据不同的dialect进行转换（例如oracle中关于limit的实现跟其他数据库不一样）

sqlOrderBy
```java
public void unparse(
        SqlWriter writer,
        SqlCall call,
        int leftPrec,
        int rightPrec) {
      SqlOrderBy orderBy = (SqlOrderBy) call;
      final SqlWriter.Frame frame =
          writer.startList(SqlWriter.FrameTypeEnum.ORDER_BY);
      // 等价于直接通过sqlSelect去转换
      orderBy.query.unparse(writer, getLeftPrec(), getRightPrec());
      if (orderBy.orderList != SqlNodeList.EMPTY) {
        writer.sep(getName());
        writer.list(SqlWriter.FrameTypeEnum.ORDER_BY_LIST, SqlWriter.COMMA,
            orderBy.orderList);
      }
      if (orderBy.offset != null) {
        final SqlWriter.Frame frame2 =
            writer.startList(SqlWriter.FrameTypeEnum.OFFSET);
        writer.newlineAndIndent();
        writer.keyword("OFFSET");
        orderBy.offset.unparse(writer, -1, -1);
        writer.keyword("ROWS");
        writer.endList(frame2);
      }
      if (orderBy.fetch != null) {
        final SqlWriter.Frame frame3 =
            writer.startList(SqlWriter.FrameTypeEnum.FETCH);
        writer.newlineAndIndent();
        writer.keyword("FETCH");
        writer.keyword("NEXT");
        orderBy.fetch.unparse(writer, -1, -1);
        writer.keyword("ROWS");
        writer.keyword("ONLY");
        writer.endList(frame3);
      }
      writer.endList(frame);
    }
```

通过sqlSelect转换，然后写死了fetch部分....


#### 修改

修改SqlOrderBy的源码,将offset和fetch的部分，改为通过write的fetchOffset的方法（内部会通过自身的方言来实现）

SqlPrettyWriter.fetchOffset
```java
public void fetchOffset(SqlNode fetch, SqlNode offset) {
    if (fetch == null && offset == null) {
      return;
    }
    dialect.unparseOffsetFetch(this, offset, fetch);
  }
```

SqlOrderBy源码修改部分
```java
 public void unparse(
                SqlWriter writer,
                SqlCall call,
                int leftPrec,
                int rightPrec) {
            SqlOrderBy orderBy = (SqlOrderBy) call;
            final SqlWriter.Frame frame =
                    writer.startList(SqlWriter.FrameTypeEnum.ORDER_BY);
            orderBy.query.unparse(writer, getLeftPrec(), getRightPrec());
            if (orderBy.orderList != SqlNodeList.EMPTY) {
                writer.sep(getName());
                writer.list(SqlWriter.FrameTypeEnum.ORDER_BY_LIST, SqlWriter.COMMA,
                        orderBy.orderList);
            }
            /*if (orderBy.offset != null) {
                final SqlWriter.Frame frame2 =
                        writer.startList(SqlWriter.FrameTypeEnum.OFFSET);
                writer.newlineAndIndent();
                writer.keyword("OFFSET");
                orderBy.offset.unparse(writer, -1, -1);
                writer.keyword("ROWS");
                writer.endList(frame2);
            }
            if (orderBy.fetch != null) {
                final SqlWriter.Frame frame3 =
                        writer.startList(SqlWriter.FrameTypeEnum.FETCH);
                writer.newlineAndIndent();
                writer.keyword("FETCH");
                writer.keyword("NEXT");
                orderBy.fetch.unparse(writer, -1, -1);
                writer.keyword("ROWS");
                writer.keyword("ONLY");
                writer.endList(frame3);
            }*/
            writer.fetchOffset(orderBy.fetch, orderBy.offset);
            writer.endList(frame);
        }
```

