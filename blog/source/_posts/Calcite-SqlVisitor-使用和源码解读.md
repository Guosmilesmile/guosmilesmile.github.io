---
title: Calcite SqlVisitor、RelShuttle 使用和源码解读使用和源码解读
date: 2020-09-26 16:23:04
tags:
categories: Calcite
---

## SqlVisitor

### 基础类
SqlVisitor是操作与sqlNode层面的api，代码不多，主要功能是提供访问SqlNode，如果遇到满足条件的node返回对应的操作。

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to you under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.calcite.sql.util;

import org.apache.calcite.sql.SqlCall;
import org.apache.calcite.sql.SqlDataTypeSpec;
import org.apache.calcite.sql.SqlDynamicParam;
import org.apache.calcite.sql.SqlIdentifier;
import org.apache.calcite.sql.SqlIntervalQualifier;
import org.apache.calcite.sql.SqlLiteral;
import org.apache.calcite.sql.SqlNode;
import org.apache.calcite.sql.SqlNodeList;
import org.apache.calcite.sql.SqlOperator;

/**
 * Visitor class, follows the
 * {@link org.apache.calcite.util.Glossary#VISITOR_PATTERN visitor pattern}.
 *
 * <p>The type parameter <code>R</code> is the return type of each <code>
 * visit()</code> method. If the methods do not need to return a value, use
 * {@link Void}.
 *
 * @see SqlBasicVisitor
 * @see SqlNode#accept(SqlVisitor)
 * @see SqlOperator#acceptCall
 *
 * @param <R> Return type
 */
public interface SqlVisitor<R> {
  //~ Methods ----------------------------------------------------------------

  /**
   * Visits a literal.
   *
   * @param literal Literal
   * @see SqlLiteral#accept(SqlVisitor)
   */
  R visit(SqlLiteral literal);

  /**
   * Visits a call to a {@link SqlOperator}.
   *
   * @param call Call
   * @see SqlCall#accept(SqlVisitor)
   */
  R visit(SqlCall call);

  /**
   * Visits a list of {@link SqlNode} objects.
   *
   * @param nodeList list of nodes
   * @see SqlNodeList#accept(SqlVisitor)
   */
  R visit(SqlNodeList nodeList);

  /**
   * Visits an identifier.
   *
   * @param id identifier
   * @see SqlIdentifier#accept(SqlVisitor)
   */
  R visit(SqlIdentifier id);

  /**
   * Visits a datatype specification.
   *
   * @param type datatype specification
   * @see SqlDataTypeSpec#accept(SqlVisitor)
   */
  R visit(SqlDataTypeSpec type);

  /**
   * Visits a dynamic parameter.
   *
   * @param param Dynamic parameter
   * @see SqlDynamicParam#accept(SqlVisitor)
   */
  R visit(SqlDynamicParam param);

  /**
   * Visits an interval qualifier.
   *
   * @param intervalQualifier Interval qualifier
   * @see SqlIntervalQualifier#accept(SqlVisitor)
   */
  R visit(SqlIntervalQualifier intervalQualifier);
}

```

这个接口有一个实现SqlBasicVisitor，自定义的visitor只要继承这个类，去实现需要的方法，不用所有SqlNode的类型都实现。



从上面的方法看，可以看到shuttle主要是visit每一个node，如果遇到满足条件的node原样返回。


```java

/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to you under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.calcite.sql.util;

import org.apache.calcite.sql.SqlCall;
import org.apache.calcite.sql.SqlDataTypeSpec;
import org.apache.calcite.sql.SqlDynamicParam;
import org.apache.calcite.sql.SqlIdentifier;
import org.apache.calcite.sql.SqlIntervalQualifier;
import org.apache.calcite.sql.SqlLiteral;
import org.apache.calcite.sql.SqlNode;
import org.apache.calcite.sql.SqlNodeList;

/**
 * Basic implementation of {@link SqlVisitor} which does nothing at each node.
 *
 * <p>This class is useful as a base class for classes which implement the
 * {@link SqlVisitor} interface. The derived class can override whichever
 * methods it chooses.
 *
 * @param <R> Return type
 */
public class SqlBasicVisitor<R> implements SqlVisitor<R> {
  //~ Methods ----------------------------------------------------------------

  public R visit(SqlLiteral literal) {
    return null;
  }

  public R visit(SqlCall call) {
    return call.getOperator().acceptCall(this, call);
  }

  public R visit(SqlNodeList nodeList) {
    R result = null;
    for (int i = 0; i < nodeList.size(); i++) {
      SqlNode node = nodeList.get(i);
      result = node.accept(this);
    }
    return result;
  }

  public R visit(SqlIdentifier id) {
    return null;
  }

  public R visit(SqlDataTypeSpec type) {
    return null;
  }

  public R visit(SqlDynamicParam param) {
    return null;
  }

  public R visit(SqlIntervalQualifier intervalQualifier) {
    return null;
  }

  //~ Inner Interfaces -------------------------------------------------------

  /** Argument handler.
   *
   * @param <R> result type */
  public interface ArgHandler<R> {
    /** Returns the result of visiting all children of a call to an operator,
     * then the call itself.
     *
     * <p>Typically the result will be the result of the last child visited, or
     * (if R is {@link Boolean}) whether all children were visited
     * successfully. */
    R result();

    /** Visits a particular operand of a call, using a given visitor. */
    R visitChild(
        SqlVisitor<R> visitor,
        SqlNode expr,
        int i,
        SqlNode operand);
  }

  //~ Inner Classes ----------------------------------------------------------

  /**
   * Default implementation of {@link ArgHandler} which merely calls
   * {@link SqlNode#accept} on each operand.
   *
   * @param <R> result type
   */
  public static class ArgHandlerImpl<R> implements ArgHandler<R> {
    private static final ArgHandler INSTANCE = new ArgHandlerImpl();

    @SuppressWarnings("unchecked")
    public static <R> ArgHandler<R> instance() {
      return INSTANCE;
    }

    public R result() {
      return null;
    }

    public R visitChild(
        SqlVisitor<R> visitor,
        SqlNode expr,
        int i,
        SqlNode operand) {
      if (operand == null) {
        return null;
      }
      return operand.accept(visitor);
    }
  }
}

```


### 具体实现类

#### SqlShuttle

它返回未更改的每个叶节点(returns each leaf node unchanged.)


```java

/**
 * Basic implementation of {@link SqlVisitor} which returns each leaf node
 * unchanged.
 *
 * <p>This class is useful as a base class for classes which implement the
 * {@link SqlVisitor} interface and have {@link SqlNode} as the return type. The
 * derived class can override whichever methods it chooses.
 */
public class SqlShuttle extends SqlBasicVisitor<SqlNode> {
  //~ Methods ----------------------------------------------------------------

  public SqlNode visit(SqlLiteral literal) {
    return literal;
  }

  public SqlNode visit(SqlIdentifier id) {
    return id;
  }

  public SqlNode visit(SqlDataTypeSpec type) {
    return type;
  }

  public SqlNode visit(SqlDynamicParam param) {
    return param;
  }

  public SqlNode visit(SqlIntervalQualifier intervalQualifier) {
    return intervalQualifier;
  }

  public SqlNode visit(final SqlCall call) {
    // Handler creates a new copy of 'call' only if one or more operands
    // change.
    ArgHandler<SqlNode> argHandler = new CallCopyingArgHandler(call, false);
    call.getOperator().acceptCall(this, call, false, argHandler);
    return argHandler.result();
  }

  public SqlNode visit(SqlNodeList nodeList) {
    boolean update = false;
    List<SqlNode> exprs = nodeList.getList();
    int exprCount = exprs.size();
    List<SqlNode> newList = new ArrayList<>(exprCount);
    for (SqlNode operand : exprs) {
      SqlNode clonedOperand;
      if (operand == null) {
        clonedOperand = null;
      } else {
        clonedOperand = operand.accept(this);
        if (clonedOperand != operand) {
          update = true;
        }
      }
      newList.add(clonedOperand);
    }
    if (update) {
      return new SqlNodeList(newList, nodeList.getParserPosition());
    } else {
      return nodeList;
    }
  }

.....

}

```

##### 使用方法

1. 获取select的where用到了那些key


```
select * from tableA where 1=1 date ='2020-09-26' and name in ('xx');
```
```java
 List<String> whereIdList = new ArrayList<>();
        SqlShuttle sqlShuttle =new SqlShuttle(){
            @Override
            public SqlNode visit(SqlIdentifier id) {
                whereIdList.add(id.toString());
                return super.visit(id);
            }
        };
        SqlSelect sqlSelect = (SqlSelect) sqlNode;
        sqlSelect.getWhere().accept(sqlShuttle);
```
得到 date 、name.

如果不使用这种visit,通过sqlSelect.getWhere()然后通过遍历sqlNode。
但是 1=1 会出现很大的干扰，需要自己写深搜才行。

![image](https://note.youdao.com/yws/api/personal/file/A4972CEC1C824DAF9E4A14E727226FDF?method=download&shareKey=10fe0b0171e1503d03155fb82d3b18d3)

2. 为每一个where条件加一个x。


```java
SqlShuttle sqlShuttle =new SqlShuttle(){
            @Override
            public SqlNode visit(SqlIdentifier identifier) {
                return new SqlIdentifier(identifier.names.get(0)+"x",
                        identifier.getParserPosition());
            }
        };
        SqlSelect sqlSelect = (SqlSelect) sqlNode;
        SqlNode newWhere = sqlSelect.getWhere().accept(sqlShuttle);
        sqlSelect.setWhere(newWhere);
```

sql就会变成

```
select * from tableA where 1=1 datex ='2020-09-26' and namex in ('xx');
```


#### AndFinder

判断sqlNod是否有and

```java
private static class AndFinder extends SqlBasicVisitor<Void> {
        public Void visit(SqlCall call) {
            final SqlOperator operator = call.getOperator();
            if (operator == SqlStdOperatorTable.AND) {
                throw Util.FoundOne.NULL;
            }
            return super.visit(call);
        }

        boolean containsAnd(SqlNode node) {
            try {
                node.accept(this);
                return false;
            } catch (Util.FoundOne e) {
                return true;
            }
        }
    }
    
   
```

通过抛出异常判断是否找到and。
```
 
boolean b = new AndFinder().containsAnd(sqlNode);
    
```




## RelShuttle 

RelShuttle 是操作RelNode层面，在sqlNode校验过后，进行优化的环节。


### 基础类

```java

/**
 * Visitor that has methods for the common logical relational expressions.
 */
public interface RelShuttle {
  RelNode visit(TableScan scan);

  RelNode visit(TableFunctionScan scan);

  RelNode visit(LogicalValues values);

  RelNode visit(LogicalFilter filter);

  RelNode visit(LogicalCalc calc);

  RelNode visit(LogicalProject project);

  RelNode visit(LogicalJoin join);

  RelNode visit(LogicalCorrelate correlate);

  RelNode visit(LogicalUnion union);

  RelNode visit(LogicalIntersect intersect);

  RelNode visit(LogicalMinus minus);

  RelNode visit(LogicalAggregate aggregate);

  RelNode visit(LogicalMatch match);

  RelNode visit(LogicalSort sort);

  RelNode visit(LogicalExchange exchange);

  RelNode visit(LogicalTableModify modify);

  RelNode visit(RelNode other);
}

```

存在一个基础实现RelShuttleImpl，RelHomogeneousShuttle。



