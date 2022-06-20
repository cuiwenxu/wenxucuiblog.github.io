在
以dataStream.map为例，该函数经过clean闭包处理操作后被放到StreamExecutionEnvironment类的名为transformations的集合中，该属性为protected final List<Transformation<?>> transformations = new ArrayList<>();
