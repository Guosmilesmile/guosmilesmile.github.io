---
title: Calcite 带有中文引发的血案
date: 2020-09-05 16:21:04
tags:
categories: Calcite
---

```
select name,id as 编号 from table where name in ('中文');
```

### 问题一

as 中文  会报错

```
Caused by: org.apache.calcite.runtime.CalciteException: Failed to encode '中文' in character set 'ISO-8859-1'
```

#### 解决方法

在项目的资源文件新建一个saffron.properties文件

内容如下

```
calcite.default.charset = utf8
```
然后在org.apache.calcite.config.CalciteSystemProperty#loadProperties函数打断点查看是否加载该配置文件即可



### 问题二



where 条件中存在中文，转出来的语句会变成

如果是默认编码
```
select name,id as 编号 from table where name in (u'\4e2d\6587');
```

如果是utf8编码

```
select name,id as 编号 from table where name in (_UTF8'中文');
```

我们转出去的sql是不需要_UTF8前缀的。


#### 解决方法

在sqlNode的toString方法，默认是使用AnsiSqlDialect作为标准方言。


原先代码SqlDialect

```java


/** Appends a string literal to a buffer.
   *
   * @param buf Buffer
   * @param charsetName Character set name, e.g. "utf16", or null
   * @param val String value
   */
  public void quoteStringLiteral(StringBuilder buf, String charsetName,
      String val) {
    if (containsNonAscii(val) && charsetName == null) {
      quoteStringLiteralUnicode(buf, val);
    } else {
      if (charsetName != null) {
        buf.append("_");
        buf.append(charsetName);
      }
      buf.append(literalQuoteString);
      buf.append(val.replace(literalEndQuoteString, literalEscapedQuote));
      buf.append(literalEndQuoteString);
    }
  }
```

修改源码AnsiSqlDialect.java

重载quoteStringLiteral方法
```java



/** Appends a string literal to a buffer.
   *
   * @param buf Buffer
   * @param charsetName Character set name, e.g. "utf16", or null
   * @param val String value
   */
  @Override
  public void quoteStringLiteral(StringBuilder buf, String charsetName,
      String val) {
   
      if (charsetName != null) {
        buf.append("_");
        buf.append(charsetName);
      }
      buf.append(literalQuoteString);
      buf.append(val.replace(literalEndQuoteString, literalEscapedQuote));
      buf.append(literalEndQuoteString);
    
  }

```

