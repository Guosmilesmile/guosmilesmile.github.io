---
title: Flink 自定义metric监控
date: 2019-12-14 16:57:52
tags:
categories:
	- Flink
---

![image](https://note.youdao.com/yws/api/personal/file/2276E0D9C1224D38A1B425332D1AA684?method=download&shareKey=9fe64a99acbc16490a3133997f2493e3)

### 直接上报es
```java
package com.metric;

import com.alibaba.fastjson.JSON;
import com.metric.entity.AbstractReporter;
import com.metric.entity.MeasurementInfo;
import com.metric.entity.MeasurementInfoProvider;
import com.metric.system.ConstConfig;
import com.metric.utils.*;
import org.apache.flink.metrics.Gauge;
import org.apache.flink.metrics.Meter;
import org.apache.flink.metrics.MetricConfig;
import org.apache.flink.metrics.reporter.Scheduled;
import org.elasticsearch.action.bulk.BulkProcessor;
import org.elasticsearch.action.index.IndexRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.*;


public class FlinkReport extends AbstractReporter<MeasurementInfo> implements Scheduled {

    private static final Logger LOGGER = LoggerFactory.getLogger(FlinkReport.class);
    private BulkProcessor bulkProcessor;
    private Set<String> gaugesSet = new HashSet<>();

    public FlinkReport() {
        super(new MeasurementInfoProvider());
    }

    @Override
    public void open(MetricConfig config) {
        InitUtils.initSpringContext();
        EsClient esClient = SpringContextUtil.getBean(EsClient.class);
        bulkProcessor = ESUtil.getBulkProcessor(esClient.getClient());
        gaugesSet.add("outPoolUsage");
        gaugesSet.add("inPoolUsage");
    }


    @Override
    public void report() {
        try {
            //获取index名称
            String index = ConstConfig.ES_INDEX;
            Instant timestamp = Instant.now();
            long time = timestamp.toEpochMilli() / 1000 / 10 * 10 * 1000;
            String date = TimeUtil.changeLong2String(time, "yyyy-MM-dd");
            Map<String, Object> map = new HashMap<>();
            for (Map.Entry<Gauge<?>, MeasurementInfo> entry : gauges.entrySet()) {
                MeasurementInfo measurementInfo = entry.getValue();
                if (!gaugesSet.contains(measurementInfo.getName())) {
                    continue;
                }
                map.put("value", entry.getKey().getValue());
                map.put("timeStamp", time);
                map.put("metric_name", measurementInfo.getName());
                Map<String, String> tags = measurementInfo.getTags();
                map.putAll(tags);
                String jsonLine = JSON.toJSONString(map);
                this.bulkProcessor.add(new IndexRequest(index + "_" + date, ConstConfig.ES_TYPE).source(jsonLine));
            }
            map.clear();

            for (Map.Entry<Meter, MeasurementInfo> entry : meters.entrySet()) {
                map.put("value", entry.getKey().getCount());
                map.put("timeStamp", time);
                map.put("rate", entry.getKey().getRate());
                MeasurementInfo measurementInfo = entry.getValue();
                map.put("metric_name", measurementInfo.getName());
                Map<String, String> tags = measurementInfo.getTags();
                map.putAll(tags);
                String jsonLine = JSON.toJSONString(map);
                this.bulkProcessor.add(new IndexRequest(index + "_" + date, ConstConfig.ES_TYPE).source(jsonLine));
            }
            map.clear();
        } catch (ConcurrentModificationException | NoSuchElementException e) {
            // ignore - may happen when metrics are concurrently added or removed
            // report next time
            return;
        }
        this.bulkProcessor.flush();
    }


    @Override
    public void close() {
        this.bulkProcessor.flush();
        this.bulkProcessor.close();
    }
}

```

### 直接上报kafka

```java
package com.metric;

import com.alibaba.fastjson.JSON;
import org.apache.flink.metrics.Gauge;
import org.apache.flink.metrics.Meter;
import org.apache.flink.metrics.MetricConfig;
import org.apache.flink.metrics.reporter.Scheduled;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.time.Instant;
import java.util.*;


public class FlinkReportToKafka extends AbstractReporter<MeasurementInfo> implements Scheduled {

    private static final Logger LOGGER = LoggerFactory.getLogger(FlinkReportToKafka.class);
    private KafkaProducer kafkaProducer;
    private String topic;
    private Set<String> gaugesSet = new HashSet<>();

    public FlinkReportToKafka() {
        super(new MeasurementInfoProvider());
    }

    @Override
    public void open(MetricConfig config) {
        InitUtils.initSpringContext();
        String kafkaServer = config.getString("kafka", "default");
        topic = config.getString("topic", "default");
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, kafkaServer);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        props.put(ProducerConfig.CLIENT_ID_CONFIG, "flink_client");
        props.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG, PropertyUtil.getString("max_size", "10485760"));
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, PropertyUtil.getString("buffer_mem", "105544320"));
        kafkaProducer = new KafkaProducer<String, String>(props);
        gaugesSet.add("outPoolUsage".toLowerCase());
        gaugesSet.add("inPoolUsage".toLowerCase());
        gaugesSet.add("inputQueueLength".toLowerCase());
        gaugesSet.add("outputQueueLength".toLowerCase());
        gaugesSet.add("avgQueueLen".toLowerCase());
    }


    @Override
    public void report() {
        try {
            //获取index名称
            Instant timestamp = Instant.now();
            long time = timestamp.toEpochMilli() / 1000 / 10 * 10 * 1000;
            Map<String, Object> map = new HashMap<>();
            for (Map.Entry<Gauge<?>, MeasurementInfo> entry : gauges.entrySet()) {
                MeasurementInfo measurementInfo = entry.getValue();
                String name = measurementInfo.getName();
                boolean find = false;
                for (String key : gaugesSet) {
                    if (name.toLowerCase().contains(key)) {
                        find = true;
                        break;
                    }
                }
                if (!find) {
                    continue;
                }
                map.put("value", entry.getKey().getValue());
                map.put("metric_name", measurementInfo.getName());
                Map<String, String> tags = measurementInfo.getTags();
                map.putAll(tags);
                map.put("timeStamp", time);
                String jsonLine = JSON.toJSONString(map);
                kafkaProducer.send(new ProducerRecord<>(topic, UUID.randomUUID().toString(), jsonLine));
            }
            map.clear();

            for (Map.Entry<Meter, MeasurementInfo> entry : meters.entrySet()) {
                MeasurementInfo measurementInfo = entry.getValue();
                map.put("value", entry.getKey().getCount());
                map.put("rate", entry.getKey().getRate());
                map.put("metric_name", measurementInfo.getName());
                Map<String, String> tags = measurementInfo.getTags();
                map.putAll(tags);
                map.put("timeStamp", time);
                String jsonLine = JSON.toJSONString(map);
                kafkaProducer.send(new ProducerRecord<>(topic, UUID.randomUUID().toString(), jsonLine));
            }
            map.clear();
        } catch (ConcurrentModificationException | NoSuchElementException e) {
            return;
        }
    }


    @Override
    public void close() {
        kafkaProducer.close();
    }
}
```
关于metric可以上报的直接，可以到flink的官网上获取，自行扩展上面的代码

https://ci.apache.org/projects/flink/flink-docs-release-1.9/monitoring/metrics.html


### 解析部分

```java

package com.metric.entity;

import org.apache.flink.metrics.Counter;
import org.apache.flink.metrics.Gauge;
import org.apache.flink.metrics.Histogram;
import org.apache.flink.metrics.Meter;
import org.apache.flink.metrics.Metric;
import org.apache.flink.metrics.MetricGroup;
import org.apache.flink.metrics.reporter.MetricReporter;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.HashMap;
import java.util.Map;


public abstract class AbstractReporter<MetricInfo> implements MetricReporter {
	protected final Logger log = LoggerFactory.getLogger(getClass());

	protected final Map<Gauge<?>, MetricInfo> gauges = new HashMap<>();
	protected final Map<Counter, MetricInfo> counters = new HashMap<>();
	protected final Map<Histogram, MetricInfo> histograms = new HashMap<>();
	protected final Map<Meter, MetricInfo> meters = new HashMap<>();
	protected final MetricInfoProvider<MetricInfo> metricInfoProvider;

	protected AbstractReporter(MetricInfoProvider<MetricInfo> metricInfoProvider) {
		this.metricInfoProvider = metricInfoProvider;
	}

	@Override
	public void notifyOfAddedMetric(Metric metric, String metricName, MetricGroup group) {
		final MetricInfo metricInfo = metricInfoProvider.getMetricInfo(metricName, group);
		synchronized (this) {
			if (metric instanceof Counter) {
				counters.put((Counter) metric, metricInfo);
			} else if (metric instanceof Gauge) {
				gauges.put((Gauge<?>) metric, metricInfo);
			} else if (metric instanceof Histogram) {
				histograms.put((Histogram) metric, metricInfo);
			} else if (metric instanceof Meter) {
				meters.put((Meter) metric, metricInfo);
			} else {
				log.warn("Cannot add unknown metric type {}. This indicates that the reporter " +
					"does not support this metric type.", metric.getClass().getName());
			}
		}
	}

	@Override
	public void notifyOfRemovedMetric(Metric metric, String metricName, MetricGroup group) {
		synchronized (this) {
			if (metric instanceof Counter) {
				counters.remove(metric);
			} else if (metric instanceof Gauge) {
				gauges.remove(metric);
			} else if (metric instanceof Histogram) {
				histograms.remove(metric);
			} else if (metric instanceof Meter) {
				meters.remove(metric);
			} else {
				log.warn("Cannot remove unknown metric type {}. This indicates that the reporter " +
					"does not support this metric type.", metric.getClass().getName());
			}
		}
	}
}

```

```java

package com.metric.entity;

import java.util.Map;

public final class MeasurementInfo {
    private final String name;
    private final Map<String, String> tags;

    MeasurementInfo(String name, Map<String, String> tags) {
        this.name = name;
        this.tags = tags;
    }

    public String getName() {
        return name;
    }

    public Map<String, String> getTags() {
        return tags;
    }

    @Override
    public String toString() {
        StringBuilder stringBuilder = new StringBuilder("name:" + name);
        for (Map.Entry<String, String> entry : tags.entrySet()) {
            stringBuilder.append(" key:" + entry.getKey() + " value: " + entry.getValue() + System.lineSeparator());
        }
        return stringBuilder.toString();
    }
}

```

```java

package com.metric.entity;

import org.apache.flink.annotation.VisibleForTesting;
import org.apache.flink.metrics.CharacterFilter;
import org.apache.flink.metrics.MetricGroup;
import org.apache.flink.runtime.metrics.groups.AbstractMetricGroup;
import org.apache.flink.runtime.metrics.groups.FrontMetricGroup;

import java.util.HashMap;
import java.util.Map;
import java.util.regex.Pattern;

public class MeasurementInfoProvider implements MetricInfoProvider<MeasurementInfo> {
    @VisibleForTesting
    static final char SCOPE_SEPARATOR = '_';

    private static final CharacterFilter CHARACTER_FILTER = new CharacterFilter() {
        private final Pattern notAllowedCharacters = Pattern.compile("[^a-zA-Z0-9:_]");

        @Override
        public String filterCharacters(String input) {
            return notAllowedCharacters.matcher(input).replaceAll("_");
        }
    };

    public MeasurementInfoProvider() {
    }

    @Override
    public MeasurementInfo getMetricInfo(String metricName, MetricGroup group) {
        return new MeasurementInfo(getScopedName(metricName, group), getTags(group));
    }

    private static Map<String, String> getTags(MetricGroup group) {
        // Keys are surrounded by brackets: remove them, transforming "<name>" to "name".
        Map<String, String> tags = new HashMap<>();
        for (Map.Entry<String, String> variable : group.getAllVariables().entrySet()) {
            String name = variable.getKey();
            tags.put(name.substring(1, name.length() - 1), variable.getValue());
        }
        return tags;
    }

	/**
	 * only return the metricName
	 * @param metricName
	 * @param group
	 * @return
	 */
    private static String getScopedName(String metricName, MetricGroup group) {
		return getLogicalScope(group) + SCOPE_SEPARATOR + metricName;
    }

    private static String getLogicalScope(MetricGroup group) {
        return ((FrontMetricGroup<AbstractMetricGroup<?>>) group).getLogicalScope(CHARACTER_FILTER, SCOPE_SEPARATOR);
    }
}


```

```java

package com.metric.entity;

import org.apache.flink.metrics.MetricGroup;

/**
 * A generic interface to provide custom information for metrics. See {@link AbstractReporter}.
 *
 * @param <MetricInfo> Custom metric information type
 */
public interface MetricInfoProvider<MetricInfo> {

	MetricInfo getMetricInfo(String metricName, MetricGroup group);
}

```

### 配置项：

```
    metrics.reporters: flinkreport
    metrics.reporter.flinkreport.class: com.metric.FlinkReportToKafka
    metrics.reporter.flinkreport.kafka: kafka-address:8888
    metrics.reporter.flinkreport.topic: flink_metric

```


### 遇到的坑


在上报kafka的时候，监控配置的pom使用的kafka版本和项目使用的kafka版本是一直的，可是如果在监控程序中打入flink-kafka的依赖，程序中也打入该依赖，会出现依赖冲突，导致程序无法使用kafkaConsume。


解决方法是将kafka的依赖提取，统一放入Flink平台的lib包中。只是这种方法需要固定对接的kafka版本，1.0.0以上的kafka均无碍。遇到1.0.0以下的版本需要另外打包依赖。



### todo

当时在metric的程序依赖中打的是flink-kafka的依赖，如果将依赖只打kafkaClient呢？ 可以尝试一下。


### 数据样例

```
 {
"_index": "flink_metric_20191214",
"_type": "flink_metric",
"_id": "AW8BTzbSgRLaxQUd3R-V",
"_score": 1,
"_source": {
"task_name": "Source: Custom Source -> Flat Map -> bundle function",
"task_attempt_num": "0",
"timeStamp": 1576274870000,
"task_attempt_id": "25d6acd2a5460a6b7405ab2ea16f38e8",
"metric_name": "taskmanager_job_task_buffers_outputQueueLength",
"job_name": "PlayerCountFlink",
"tm_id": "cab63a63a32f4ca107908b4a5445fb3c",
"job_id": "0ad7f90fc5bba9ab4b08567e9e1cab12",
"host": "flink-taskmanager-9457b74bb-562hx",
"task_id": "cbc357ccb763df2852fee8c4fc7d55f2",
"value": 90,
"subtask_index": "2"
}
},
```
```
{
"_index": "flink_metric_20191214",
"_type": "flink_metric",
"_id": "AW8BTzbSgRLaxQUd3R-n",
"_score": 1,
"_source": {
"task_name": "Source: Custom Source -> Flat Map -> bundle function",
"task_attempt_id": "a3d864bda893be0a7f5027dfe72c53fb",
"operator_id": "72ee2076ad4244f19e7388e24679c996",
"task_id": "cbc357ccb763df2852fee8c4fc7d55f2",
"task_attempt_num": "0",
"timeStamp": 1576274870000,
"operator_name": "Sink: Unnamed",
"metric_name": "taskmanager_job_task_numRecordsOutPerSecond",
"job_name": "PlayerCountFlink",
"tm_id": "cab63a63a32f4ca107908b4a5445fb3c",
"rate": 2657.8166666666666,
"job_id": "0ad7f90fc5bba9ab4b08567e9e1cab12",
"host": "flink-taskmanager-9457b74bb-562hx",
"value": 355256995,
"subtask_index": "8"
}
}
```


模板flink-metric,es版本 1.6（无奈）

```
 {
"order": 0,
"template": "flink_metric*",
"settings": {
"index.refresh_interval": "30s",
"index.replication": "async",
"index.translog.durability": "async",
"index.translog.flush_threshold_size": "200mb",
"index.number_of_replicas": "1",
"index.routing.allocation.total_shards_per_node": "2",
"index.routing.allocation.require.tag": "hot",
"index.routing.allocation.include.group": "web1,web2,web3,web4",
"index.number_of_shards": "6"
},
"mappings": {
"flink_metric": {
"_source": {
"enabled": true
},
"properties": {
"task_name": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"task_attempt_id": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"operator_id": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"task_id": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"task_attempt_num": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"timeStamp": {
"format": "dateOptionalTime",
"type": "date",
"doc_values": true
},
"operator_name": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"metric_name": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"job_name": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"tm_id": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"rate": {
"index": "no",
"type": "double",
"doc_values": true
},
"host": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
},
"value": {
"index": "no",
"type": "double",
"doc_values": true
},
"subtask_index": {
"index": "not_analyzed",
"type": "string",
"doc_values": true
}
},
"_all": {
"enabled": false
}
}
},
"aliases": {}
}
```