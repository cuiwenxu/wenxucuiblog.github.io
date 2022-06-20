# toBitmap代码分析
要想使用bitmap做去重，第一步是要将long型数据转换成bitmap,这个功能可以实现为udf，也是以是udaf，下面是udaf版本代码分析
代码的大概含义就是调用rbm中的add方法，不停的将long数据添加到rbm中，然后将rbm转换成为二进制数组输出
```java
public class ToDorisBitmapUDAF extends AbstractGenericUDAFResolver{

    //判断
    @Override
    public GenericUDAFEvaluator getEvaluator(TypeInfo[] info)//字段的描述信息参数parameters
            throws SemanticException {
        if(info.length !=1){
            throw new UDFArgumentTypeException(info.length-1,
                    "Exactly one argument is expected.");
        }

        //返回处理逻辑的类
        return new GenericEvaluate();
    }

    public static class GenericEvaluate extends GenericUDAFEvaluator{

        protected PrimitiveObjectInspector inputOI;
        private BinaryObjectInspector inputIO2;

        //这个方法map与reduce阶段都需要执行
        /**
         * 这个方法的作用做准备工作，在四种mode的情况下都会调用这个方法，准备工作一般是指定各阶段的输入和输出
         *
         * 这里的代码是指定map reduce阶段的输入，hive udaf会有输入参数，但是hive并不知道其数据类型，所以要在这里显式的指定
         * 此外：
         * map阶段：parameters长度与udaf输入的参数个数有关
         * reduce阶段：parameters长度为1
         */
        //初始化
        @Override
        public ObjectInspector init(Mode m, ObjectInspector[] parameters)
                throws HiveException {
            super.init(m, parameters);
            //除了map阶段的输出之外，其他的输入都是二进制的
            if (m == Mode.PARTIAL1) {
                this.inputOI = (PrimitiveObjectInspector)parameters[0];
            }else{
                this.inputIO2 = (BinaryObjectInspector) parameters[0];
            }

            return PrimitiveObjectInspectorFactory.javaByteArrayObjectInspector;
        }

        //map阶段
        @Override
        public void iterate(AggregationBuffer agg, Object[] parameters)//bitmapInter 缓存结果
                throws HiveException {

            assert(parameters.length==1);

            if(parameters==null || parameters[0]==null || parameters[0].equals("") ){
                return;
            }
            Long input =PrimitiveObjectInspectorUtils.getLong(parameters[0],this.inputOI);

            if( input == null){
                return;
            }

            ((BitmapAgg) agg).bitmap.add(input);
        }

        @AggregationType(estimable = true)
        static class BitmapAgg extends AbstractAggregationBuffer {
            RBitmapDoris bitmap;
            @Override
            public int estimate() {
                // 缓存中存放的是long，所以这里占用的存储空间预估为8byte，64位
                return JavaDataModel.PRIMITIVES2;
            }
        }

        //获得一个聚合的缓冲对象，每个map执行一次
        @Override
        public AbstractAggregationBuffer getNewAggregationBuffer() throws HiveException {
            BitmapAgg bitmapAgg = new BitmapAgg();
            reset(bitmapAgg);
            return bitmapAgg;
        }

        //重置
        @Override
        public void reset(AggregationBuffer agg) throws HiveException {
            BitmapAgg bitmapAgg = (BitmapAgg) agg;
            if(bitmapAgg.bitmap==null){
                bitmapAgg.bitmap= new RBitmapDoris();
            }
        }

        //该方法当做iterate执行后，部分结果返回。
        @Override
        public Object terminatePartial(AggregationBuffer agg)
                throws HiveException {
            return ((BitmapAgg) agg).bitmap.toSerializedByte();
        }

        //在combine和reduce阶段会调用merge方法，方法里将二进制数组添加到rbm中，做取并集操作
        @Override
        public void merge(AggregationBuffer agg, Object partial)
                throws HiveException {
            if(partial != null){
                byte[] input= this.inputIO2.getPrimitiveJavaObject(partial);
                ((BitmapAgg) agg).bitmap.addSerializedByte(input);
            }
        }

        //返回最后的输出结果
        @Override
        public Object terminate(AggregationBuffer agg) throws HiveException {
            return ((BitmapAgg) agg).bitmap.toSerializedByte();
        }
    }
}

```
