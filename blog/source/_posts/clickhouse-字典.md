---
title: clickhouse 字典
date: 2019-04-02 00:10:58
tags:
categories: clickhouse
---



### 介绍
一个字典是一个映射 (key -> attributes) ，能够作为函数被用于查询。你可以认为它可以作为更便捷和高效的 JOIN 类型，作为维度表。

数据字典有两种，一个是内置字典，另一个是外置字典。

### 外部字典
当启动服务器时，字典能够被创建，或者初次使用。通过dictionaries_lazy_load参数被定义，在主服务器的配置文件中。这个参数是可选的，默认情况是’true’，每个字典初次使用时被创建。如果词典创建失败，正在使用词典的函数抛出一个异常。如果是’false’，当服务器启动时，所有的字典被创建，如果有一个错误，那么服务器将停机。

#### 支持类型
* 文件 file
* mysql
* clickhouse
#### config.xml
```shell
<dictionaries_config>*_dictionary.xml</dictionaries_config>
```

#### dictionary.xml

```xml
<comment>Optional element with any content; completely ignored.</comment>

<!–You can set any number of different dictionaries. -->

<dictionary>

<!-- Dictionary name. The dictionary will be accessed for use by this name. -->
<!--字典名称-->
<name>os</name>

<!-- Data source. -->
<!-- 数据源配置  选一种 -->
<source>

    <!-- Source is a file in the local file system. -->
    <!-- 本地文件或者是文件系统 -->
    <file>
    
        <!-- Path on the local file system. -->
        <path>/opt/dictionaries/os.tsv</path>
    
        <!-- Which format to use for reading the file. -->
        <format>TabSeparated</format>
    
    </file>
    
    <!-- or the source is a table on a MySQL server.-->
    <!-- mysql数据源 -->
    <mysql>
    
        <!- - These parameters can be specified outside (common for all replicas) or inside a specific replica - ->
        <port>3306</port>
        
        <user>clickhouse</user>
        
        <password>qwerty</password>
        
        <!- - Specify from one to any number of replicas for fault tolerance. - ->
    
        <replica>
        
        <host>example01-1</host>
        
        <priority>1</priority> <!- - The lower the value, the higher the priority. - ->
        
        </replica>
        
        <replica>
        
        <host>example01-2</host>
        
        <priority>1</priority>
        
        </replica>
        
        <db>conv_main</db>
        
        <table>counters</table>
    
        <!--可选-->
        <where>name not like 'xx_%'</where>
        
    </mysql>
    
    
    <!-- or the source is a table on the ClickHouse server.-->
    <!-- clickhouse 数据源-->
    <clickhouse>
    
        <host>example01-01-1</host>
        
        <port>9000</port>
        
        <user>default</user>
        
        <password></password>
        
        <db>default</db>
        
        <table>counters</table>
        
        <!--可选-->
        <where></where>
        
        <!- - If the address is similar to localhost, the request is made without network interaction. For fault tolerance, you can create a Distributed table on localhost and enter it. - ->
    
        <!-- 如果address 设为localhost，可以减少网络io -->
    
    </clickhouse>

</source>


<!-- Update interval for fully loaded dictionaries. 0 - never update. -->
<!-- 更新字典的时间间隔，如果设为0则为不更新-->

<lifetime>
    
    <min>300</min>
    
    <max>360</max>


<!-- The update interval is selected uniformly randomly between min and max, in order to spread out the load when updating dictionaries on a large number of servers. -->
<!-- 更新时间选择随机在最大时间与最小时间之间，避免服务器大规模更新字典导致-->
</lifetime>    

<!-- or <!- - The update interval for fully loaded dictionaries or invalidation time for cached dictionaries. 0 - never update. 
<!-- 
<lifetime>300</lifetime>
-->


<!-- Method for storing in memory. -->
<!-- 存储方式 -->
<layout>
    <complex_key_hashed/>
    
    <!--
        <flat/>
        
        or
         
        <hashed />
        
        or
        
        <cache>
        
        <!- - Cache size in number of cells; rounded up to a degree of two. - ->
        
        <size_in_cells>1000000000</size_in_cells>
        
        </cache>
        
        or
        
        <ip_trie />
    
    –>
</layout>

<!-- Structure. -->
<!-- 数据结构 -->
<structure>

<!-- Description of the column that serves as the dictionary identifier (key). -->

<!-- Description of the column that serves as the dictionary identifier (key). -->

    <key>
    
    <!-- Column name with ID. -->
    
        <attribute>
                <name>ip</name>
                <type>String</type>
        </attribute>

    
    </key>
    
    <attribute>
    
        <!-- Column name. -->
    
        <name>Name</name>
        
        <!-- Column type. (How the column is understood when loading. For MySQL, a table can have TEXT, VARCHAR, and BLOB, but these are all loaded as String) -->
        
        <type>String</type>
        
        <!-- Value to use for a non-existing element. In the example, an empty string. -->
        
        <null_value></null_value>
    
    
    </attribute>
    
    <!-- 可以配置多个属性，调用的时候选择对应的attribute-->
    <attribute>
      
        <name>sex</name>
        
        <type>String</type>
    
        <null_value></null_value>
    
    </attribute>

</structure>

</dictionary>


```


#### layout配置

有6种不同的方法在内存中存储字典。
##### flat

flat是最有效的方法。如果所有的 keys 小于500,000，它正常工作。如果一个更大的 key被发现，当创建一个字典，一个异常将抛出，此字典不会被创建。字典被整体加载进入到内存中。字典使用内存量与最大的 key value 相称。限制为500,000，内存消耗不会太高。所有类型的资源都被支持。当更新时，数据(文件或者表)被整体读取。

##### Hashed

此方法比第一个效果差一点。字典也被加载到内存中，能够包含任意的条目数量（带有任意 ID）。在实际情况下，它可以用到数百万个条目，如果有足够大的内存。所有源的类型都被支持。当更新时，数据(文件或者表)被整体读取。


##### Cache

最无效的方法. 如果字典不适合在内存，它是合适的. 它是确定数据槽位数量的缓存，可将频繁访问的数据放在这. MySQL, ClickHouse, Executable, HTTP 资源都被支持, 但是文件资源不被支持. 当在一个词典中搜索时, 缓存是第一个被搜索的. 对于每一个数据块, 所有的键没有在缓存中找到的 (或者已经超时) 被收集在一个包中, 被发送到带有查询的源端 SELECT attrs…FROM db.table WHERE id IN(k1,k2,…). 接收的数据被写入到缓存中.

##### Range_hashed
太复杂，先忽略。。

##### complex_key_hashed

对于复杂的 keys，与hashed是相同的，

##### complex_key_cache

对于复杂的 keys，与 cache 是相同的。



#### 带有复杂Keys的字典

你能够使用tuples，由任意类型的数据域作为 keys 组成。配置你的字典-带有complex_key_hashed或者complex_key_cache配置。

Key 结构被配置不在<id>元素中，而在<key>元素中。key tuple 的数据域被配置与字典属性类似。例如：

```
<structure>

<key>

<attribute>
    
    <name>field1</name>
    
    <type>String</type>

</attribute>

<attribute>

    <name>field2</name>
    
    <type>UInt32</type>

</attribute>

```
当使用这些字典时, 使用 Tuple 域值作为 一个 key，在 dictGet* 函数中. Example:dictGetString(‘dict_name’,‘attr_name’,tuple(‘field1_value’,123)).


#### 字典的使用

```

dictGetString(
		'字典名称',
		'需要获取的属性，如果存在多属性',
		tuple(入参字段)
	) 

```
```sql
dictGetT(‘dict_name’,‘attr_name’,tuple(ip))
```



http://www.clickhouse.com.cn/topic/5a3bb6862141c2917483556c

https://blog.csdn.net/bluetjs/article/details/80322497