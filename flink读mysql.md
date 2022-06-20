flink读取mysql需要继承RichSourceFunction，代码如下
```java
import com.aibee.flinktemplate.model.GateEvent;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.source.RichSourceFunction;

import java.sql.*;

public class FaceEventMysqlSource extends RichSourceFunction<GateEvent> {


    PreparedStatement ps;
    private Connection connection;
    boolean isRunning = true;
    long timeInterval = 30 * 60 * 1000L;  // 间隔 1h 查询一次

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        connection = getConnection();
        String sql = "select id,site_id,gate_id,`timestamp` from gate_event;";
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

    /**
     * 在run方法中读取数据库
     * @param ctx
     * @throws Exception
     */
    @Override
    public void run(SourceContext<GateEvent> ctx) throws Exception {
        while (isRunning) {
            ResultSet resultSet = ps.executeQuery();
            while (resultSet.next()) {
                GateEvent gateEvent = new GateEvent(
                        resultSet.getString("id"),
                        resultSet.getString("site_id").trim(),
                        resultSet.getString("gate_id").trim(),
                        resultSet.getString("timestamp"));
                ctx.collect(gateEvent);
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
            con = DriverManager.getConnection("jdbc:mysql://pm:3306/aibee_di?useUnicode=true&characterEncoding=UTF-8", "", "");
        } catch (Exception e) {
            System.out.println("-----------mysql get connection has exception , msg = " + e.getMessage());
        }
        return con;
    }

}

```
测试上段代码
```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

public class TestMysqlSource {

    public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        env.addSource(new FaceEventMysqlSource()).map(x->x.getSite_id()).print();
                //.print();

        env.execute("Flink add data sourc");
    }

}
```
