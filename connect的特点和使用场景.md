### connect的特点和使用场景
- 1、ConnectedStreams 连接的两个流类型可以不一致，比如其中之一为string,另一个可以为int。
- 2、ConnectedStreams 会对两个流的数据应用不同的处理方法，==并且双流之间可以共享状态。这在第一个流的输入会影响第二个流时, 会非常有用==。

*for example：计算客流添加营业时间过滤*
其中一个流是kafka客流数据，其中包含客户到访时间，另一个流是读mysql中营业时间配置表生成的
```java
DataStream<BusinessTime> businessTimeDataStream = env.addSource(new BusinessTimeSource()).setParallelism(1).broadcast();
```
完整代码如下
```java
import com.aibee.common.source.KafkaSourceJava;
import com.aibee.mall.model.BusinessTime;
import com.aibee.mall.model.FaceEvent;
import com.aibee.mall.sink.FilterTimeSink;
import com.aibee.mall.source.BusinessTimeSource;
import com.alibaba.fastjson.JSON;
import com.alibaba.fastjson.JSONObject;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoFlatMapFunction;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.util.Collector;


public class FilterTimeTest {


    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        //kafka客流
        ParameterTool parameter = ParameterTool.fromArgs(args);
        FlinkKafkaConsumer<String> flinkKafkaConsumer = new KafkaSourceJava(parameter).getConsumer(parameter.getRequired("group_id"));
        DataStream<String> kafkaConsumerDataStream = env.addSource(flinkKafkaConsumer).setParallelism(5);
        //读取营业时间的配置流，设置并行度为1，并且broadcast
        DataStream<BusinessTime> businessTimeDataStream = env.addSource(new BusinessTimeSource()).setParallelism(1).broadcast();

        kafkaConsumerDataStream
                .map(new RichMapFunction<String, FaceEvent>() {
                    @Override
                    public FaceEvent map(String value) throws Exception {
                        JSONObject faceEventStr = JSON.parseObject(value);
                        return new FaceEvent(faceEventStr.getString("site_id"), faceEventStr.getString("store_id")
                                , faceEventStr.getString("face_id"), faceEventStr.getString("pid")
                                , faceEventStr.getString("gender"), faceEventStr.getString("age")
                                , faceEventStr.getString("img"), faceEventStr.getString("timestamp")
                                , faceEventStr.getString("floor"));
                    }
                })
                .connect(businessTimeDataStream)
                .flatMap(new CoFlatMapFunction<FaceEvent, BusinessTime, FaceEvent>() {

                    int begin_hour, end_hour;

                    @Override
                    public void flatMap1(FaceEvent value, Collector<FaceEvent> out) throws Exception {
                        Long hour = Long.parseLong(value.timestamp) / 1000 / 60 / 60 % 24 + 8;
                        System.out.println(value.timestamp + "-----------" + hour);
                        if (hour > begin_hour && hour < end_hour) {
                            out.collect(value);
                            System.out.println("************" + hour);
                        }
                    }

                    @Override
                    public void flatMap2(BusinessTime value, Collector<FaceEvent> out) throws Exception {
                        begin_hour = Integer.parseInt(value.getBegin_time());
                        end_hour = Integer.parseInt(value.getEnd_time());
                    }
                })
                .map(new MapFunction<FaceEvent, String>() {
                    @Override
                    public String map(FaceEvent value) throws Exception {
                        return value.site_id + "," + value.pid + "," + value.timestamp + "," + (Long.parseLong(value.timestamp) / 1000 / 60 / 60 % 24 + 8);
                    }
                })
                .addSink(new FilterTimeSink());

        env.execute();

    }


}
```
mysql source类
```java
import com.aibee.mall.model.BusinessTime;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.source.RichSourceFunction;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class BusinessTimeSource extends RichSourceFunction<BusinessTime> {

    PreparedStatement ps;
    private Connection connection;
    boolean isRunning = true;
    long timeInterval = 60 * 1000L;  // 间隔 1min 查询一次

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        connection = getConnection();
        String sql = "select begin_time,end_time from business_time_list;";
        ps = this.connection.prepareStatement(sql);
    }

    @Override
    public void close() throws Exception {
        super.close();
        if (connection != null) { //关闭连接和释放资源
            connection.close();
        }
        if (ps != null) {
            ps.close();
        }
    }

    @Override
    public void run(SourceContext<BusinessTime> ctx) throws Exception {
        while (isRunning) {
            ResultSet resultSet = ps.executeQuery();
            while (resultSet.next()) {
                BusinessTime businessTime = new BusinessTime(resultSet.getString("begin_time"), resultSet.getString("end_time"));
                ctx.collect(businessTime);
            }
            Thread.sleep(timeInterval);
        }
    }

    @Override
    public void cancel() {
        try {
            super.close();
            if (connection != null) {
                connection.close();
            }
            if (ps != null) {
                ps.close();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        isRunning = false;
    }

    private static Connection getConnection() {
        Connection con = null;
        try {
            Class.forName("com.mysql.jdbc.Driver");
            con = DriverManager.getConnection("jdbc:mysql://xxx:3306/xxx?useUnicode=true&characterEncoding=UTF-8", "", "");
        } catch (Exception e) {
            System.out.println("-----------mysql get connection has exception , msg = " + e.getMessage());
        }
        return con;
    }
}

```
