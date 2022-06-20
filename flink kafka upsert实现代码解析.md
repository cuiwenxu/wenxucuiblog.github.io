@[TOC](flink kafka upsert实现代码解析)
# 概览
本文结合源码分析Kafka upsert的实现
根据flink官方文档提供的DDL SQL解析流程图，实现自定义TableSource(or TableSink)需要做的事有三个：
1、定义可解析catalog的TableSourceFactory(or TableSinkFactory)
2、定义工厂类生产出来的product类（该product类implements DynamicSource(or DynamicSink)接口，用于调用不同数据源的connector）
3、针对不同数据源(或sink端)实现connector，如FlinkKafkaProducer、FlinkKafkaConsumer
![DDL SQL-->运行时代码流程图](https://img-blog.csdnimg.cn/20210129164732858.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)


# 代码分析
## UpsertKafkaDynamicTableFactory（工厂类）
UpsertKafkaDynamicTableFactory的功能是根据catalog配置解析出product类，所以requiredOptions、validatePKConstraints、createDynamicTableSource等类都会有
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210129164401186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
下面是带有注释的主要方法
```java
public class UpsertKafkaDynamicTableFactory implements DynamicTableSourceFactory, DynamicTableSinkFactory {
    
    //对应DDL SQL中的connector标志符
    public static final String IDENTIFIER = "upsert-kafka";

    //用于检查DDL SQL配置项是否齐全
    public Set<ConfigOption<?>> requiredOptions() {
        Set<ConfigOption<?>> options = new HashSet();
        options.add(KafkaOptions.PROPS_BOOTSTRAP_SERVERS);
        options.add(KafkaOptions.TOPIC);
        options.add(KafkaOptions.KEY_FORMAT);
        options.add(KafkaOptions.VALUE_FORMAT);
        return options;
    }

    //用于检查DDL SQL可选的配置项
    public Set<ConfigOption<?>> optionalOptions() {
        Set<ConfigOption<?>> options = new HashSet();
        options.add(KafkaOptions.KEY_FIELDS_PREFIX);
        options.add(KafkaOptions.VALUE_FIELDS_INCLUDE);
        options.add(FactoryUtil.SINK_PARALLELISM);
        return options;
    }

    //用于创建product类，即KafkaDynamicSource
    public DynamicTableSource createDynamicTableSource(Context context) {
        TableFactoryHelper helper = FactoryUtil.createTableFactoryHelper(this, context);
        ReadableConfig tableOptions = helper.getOptions();
        DecodingFormat<DeserializationSchema<RowData>> keyDecodingFormat = helper.discoverDecodingFormat(DeserializationFormatFactory.class, KafkaOptions.KEY_FORMAT);
        DecodingFormat<DeserializationSchema<RowData>> valueDecodingFormat = helper.discoverDecodingFormat(DeserializationFormatFactory.class, KafkaOptions.VALUE_FORMAT);
        helper.validateExcept(new String[]{"properties."});
        TableSchema schema = context.getCatalogTable().getSchema();
        validateTableOptions(tableOptions, keyDecodingFormat, valueDecodingFormat, schema);
        Tuple2<int[], int[]> keyValueProjections = this.createKeyValueProjections(context.getCatalogTable());
        String keyPrefix = (String)tableOptions.getOptional(KafkaOptions.KEY_FIELDS_PREFIX).orElse((Object)null);
        Properties properties = KafkaOptions.getKafkaProperties(context.getCatalogTable().getOptions());
        StartupMode earliest = StartupMode.EARLIEST;
        return new KafkaDynamicSource(schema.toPhysicalRowDataType(), keyDecodingFormat, new UpsertKafkaDynamicTableFactory.DecodingFormatWrapper(valueDecodingFormat), (int[])keyValueProjections.f0, (int[])keyValueProjections.f1, keyPrefix, KafkaOptions.getSourceTopics(tableOptions), KafkaOptions.getSourceTopicPattern(tableOptions), properties, earliest, Collections.emptyMap(), 0L, true);
    }

    //用于创建product类，即KafkaDynamicSource
    public DynamicTableSink createDynamicTableSink(Context context) {
        TableFactoryHelper helper = FactoryUtil.createTableFactoryHelper(this, context);
        ReadableConfig tableOptions = helper.getOptions();
        EncodingFormat<SerializationSchema<RowData>> keyEncodingFormat = helper.discoverEncodingFormat(SerializationFormatFactory.class, KafkaOptions.KEY_FORMAT);
        EncodingFormat<SerializationSchema<RowData>> valueEncodingFormat = helper.discoverEncodingFormat(SerializationFormatFactory.class, KafkaOptions.VALUE_FORMAT);
        helper.validateExcept(new String[]{"properties."});
        TableSchema schema = context.getCatalogTable().getSchema();
        validateTableOptions(tableOptions, keyEncodingFormat, valueEncodingFormat, schema);
        Tuple2<int[], int[]> keyValueProjections = this.createKeyValueProjections(context.getCatalogTable());
        String keyPrefix = (String)tableOptions.getOptional(KafkaOptions.KEY_FIELDS_PREFIX).orElse((Object)null);
        Properties properties = KafkaOptions.getKafkaProperties(context.getCatalogTable().getOptions());
        Integer parallelism = (Integer)tableOptions.get(FactoryUtil.SINK_PARALLELISM);
        return new KafkaDynamicSink(schema.toPhysicalRowDataType(), keyEncodingFormat, new UpsertKafkaDynamicTableFactory.EncodingFormatWrapper(valueEncodingFormat), (int[])keyValueProjections.f0, (int[])keyValueProjections.f1, keyPrefix, (String)((List)tableOptions.get(KafkaOptions.TOPIC)).get(0), properties, (FlinkKafkaPartitioner)null, KafkaSinkSemantic.AT_LEAST_ONCE, true, parallelism);
    }

    //用于验证DDL SQL配置项
    private static void validateTableOptions(ReadableConfig tableOptions, Format keyFormat, Format valueFormat, TableSchema schema) {
        validateTopic(tableOptions);
        validateFormat(keyFormat, valueFormat, tableOptions);
        validatePKConstraints(schema);
    }

    //用于验证topic
    private static void validateTopic(ReadableConfig tableOptions) {
        List<String> topic = (List)tableOptions.get(KafkaOptions.TOPIC);
        if (topic.size() > 1) {
            throw new ValidationException("The 'upsert-kafka' connector doesn't support topic list now. Please use single topic as the value of the parameter 'topic'.");
        }
    }
    
    //用于验证pk,upsert模式下必须要有pk
    private static void validatePKConstraints(TableSchema schema) {
        if (!schema.getPrimaryKey().isPresent()) {
            throw new ValidationException("'upsert-kafka' tables require to define a PRIMARY KEY constraint. The PRIMARY KEY specifies which columns should be read from or write to the Kafka message key. The PRIMARY KEY also defines records in the 'upsert-kafka' table should update or delete on which keys.");
        }
    }
    
    //...省略部分代码
    
}
```
## KafkaDynamicSource（product类）
product类，实现ScanTableSource接口。功能是创建各数据源的connector，并暴露给flink Runtime使用，所以主要的方法为createKafkaConsumer(创建connector)、getScanRuntimeProvider
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210129165529471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
```java
public class KafkaDynamicSource implements ScanTableSource, SupportsReadingMetadata, SupportsWatermarkPushDown {

    //暴露connector给flink runtime
    public ScanRuntimeProvider getScanRuntimeProvider(ScanContext context) {
        DeserializationSchema<RowData> keyDeserialization = this.createDeserialization(context, this.keyDecodingFormat, this.keyProjection, this.keyPrefix);
        DeserializationSchema<RowData> valueDeserialization = this.createDeserialization(context, this.valueDecodingFormat, this.valueProjection, (String)null);
        TypeInformation<RowData> producedTypeInfo = context.createTypeInformation(this.producedDataType);
        FlinkKafkaConsumer<RowData> kafkaConsumer = this.createKafkaConsumer(keyDeserialization, valueDeserialization, producedTypeInfo);
        return SourceFunctionProvider.of(kafkaConsumer, false);
    }

    public Map<String, DataType> listReadableMetadata() {
        Map<String, DataType> metadataMap = new LinkedHashMap();
        this.valueDecodingFormat.listReadableMetadata().forEach((key, value) -> {
            DataType var10000 = (DataType)metadataMap.put("value." + key, value);
        });
        Stream.of(KafkaDynamicSource.ReadableMetadata.values()).forEachOrdered((m) -> {
            DataType var10000 = (DataType)metadataMap.putIfAbsent(m.key, m.dataType);
        });
        return metadataMap;
    }

    //创建connector
    protected FlinkKafkaConsumer<RowData> createKafkaConsumer(DeserializationSchema<RowData> keyDeserialization, DeserializationSchema<RowData> valueDeserialization, TypeInformation<RowData> producedTypeInfo) {
        MetadataConverter[] metadataConverters = (MetadataConverter[])this.metadataKeys.stream().map((k) -> {
            return (KafkaDynamicSource.ReadableMetadata)Stream.of(KafkaDynamicSource.ReadableMetadata.values()).filter((rm) -> {
                return rm.key.equals(k);
            }).findFirst().orElseThrow(IllegalStateException::new);
        }).map((m) -> {
            return m.converter;
        }).toArray((x$0) -> {
            return new MetadataConverter[x$0];
        });
        boolean hasMetadata = this.metadataKeys.size() > 0;
        int adjustedPhysicalArity = this.producedDataType.getChildren().size() - this.metadataKeys.size();
        int[] adjustedValueProjection = IntStream.concat(IntStream.of(this.valueProjection), IntStream.range(this.keyProjection.length + this.valueProjection.length, adjustedPhysicalArity)).toArray();
        KafkaDeserializationSchema<RowData> kafkaDeserializer = new DynamicKafkaDeserializationSchema(adjustedPhysicalArity, keyDeserialization, this.keyProjection, valueDeserialization, adjustedValueProjection, hasMetadata, metadataConverters, producedTypeInfo, this.upsertMode);
        FlinkKafkaConsumer kafkaConsumer;
        if (this.topics != null) {
            kafkaConsumer = new FlinkKafkaConsumer(this.topics, kafkaDeserializer, this.properties);
        } else {
            kafkaConsumer = new FlinkKafkaConsumer(this.topicPattern, kafkaDeserializer, this.properties);
        }

        switch(this.startupMode) {
        case EARLIEST:
            kafkaConsumer.setStartFromEarliest();
            break;
        case LATEST:
            kafkaConsumer.setStartFromLatest();
            break;
        case GROUP_OFFSETS:
            kafkaConsumer.setStartFromGroupOffsets();
            break;
        case SPECIFIC_OFFSETS:
            kafkaConsumer.setStartFromSpecificOffsets(this.specificStartupOffsets);
            break;
        case TIMESTAMP:
            kafkaConsumer.setStartFromTimestamp(this.startupTimestampMillis);
        }

        kafkaConsumer.setCommitOffsetsOnCheckpoints(this.properties.getProperty("group.id") != null);
        if (this.watermarkStrategy != null) {
            kafkaConsumer.assignTimestampsAndWatermarks(this.watermarkStrategy);
        }

        return kafkaConsumer;
    }

}
```
## FlinkKafkaConsumer(FlinkKafkaProducer)（connector）
connector，真正干活的类，用于连接数据源和sink端，具体代码此处不赘述


# Q&A
> 问题：flink upsert kafka中delete 操作是给对应key发tombstone消息，但是kafka对于tombstone是定期做compact，那么在定期compact过程中下游会不会读到被delete的脏数据

解析：在DynamicKafkaSerializationSchema和DynamicKafkaDeserializationSchema这两个serde类中可找到答案

DynamicKafkaSerializationSchema序列化代码
```java
class DynamicKafkaSerializationSchema implements KafkaSerializationSchema<RowData>, KafkaContextAware<RowData> {
    
    public ProducerRecord<byte[], byte[]> serialize(RowData consumedRow, @Nullable Long timestamp) {
    
            //省略部分代码
            //序列化过程中在upsert模式下，如果执行的操作为RowKind.DELETE或RowKind.UPDATE_BEFORE                 
            //则将其value置为null（对应kafka的墓碑消息）
            if (this.upsertMode) {
                if (kind != RowKind.DELETE && kind != RowKind.UPDATE_BEFORE) {
                    valueRow.setRowKind(RowKind.INSERT);
                    valueSerialized = this.valueSerialization.serialize(valueRow);
                } else {
                    valueSerialized = null;
                }
            } else {
                valueSerialized = this.valueSerialization.serialize(valueRow);
            }
            
            return new ProducerRecord(this.topic, this.extractPartition(consumedRow, keySerialized, valueSerialized), (Long)this.readMetadata(consumedRow, WritableMetadata.TIMESTAMP), keySerialized, valueSerialized, (Iterable)this.readMetadata(consumedRow, WritableMetadata.HEADERS));

            //省略部分代码
            
    }

}

```
DynamicKafkaDeserializationSchema反序列化代码
```java
class DynamicKafkaDeserializationSchema implements KafkaDeserializationSchema<RowData> {
    

    public void deserialize(ConsumerRecord<byte[], byte[]> record, Collector<RowData> collector) throws Exception {
       
            //在upsert模式下，如果value为空(墓碑消息),则collect null，相当于该消息被删除
            if (record.value() == null && this.upsertMode) {
                this.outputCollector.collect((RowData)null);
            } else {
                this.valueDeserialization.deserialize((byte[])record.value(), this.outputCollector);
            }

            this.keyCollector.buffer.clear();
        }
    }
   
}

```
__经过以上分析，upsert的写或读都必须使用特定的api，简单的建一个consumer大概率会读到脏数据（受kafka compact周期的影响，compact执行前或者执行过程中会读到delete的脏数据）__


