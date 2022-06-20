Java代码从csv文件中读取数据，然后通过kafka包中的kafka-console-producer.sh进行写入，代码如下
- Java代码
```java
public class SourceGenerator {

    private static final long SPEED = 1000; // 每秒1000条

    public static void main(String[] args) {
        long speed = SPEED;
        if (args.length > 0) {
            speed = Long.valueOf(args[0]);
        }
        long delay = 1000_000 / speed; // 每条耗时多少毫秒

        try (InputStream inputStream = SourceGenerator.class.getClassLoader().getResourceAsStream("user_behavior.log")) {
            BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
            long start = System.nanoTime();
            while (reader.ready()) {
                String line = reader.readLine();
                System.out.println(line);

                long end = System.nanoTime();
                long diff = end - start;
                while (diff < (delay*1000)) {
                    Thread.sleep(1);
                    end = System.nanoTime();
                    diff = end - start;
                }
                start = end;
            }
            reader.close();
        } catch (IOException e) {
            throw new RuntimeException(e);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
- shell代码
```shell
source "$(dirname "$0")"/kafka-common.sh

# prepare Kafka
echo "Generating sources..."

create_kafka_topic 1 1 user_behavior
java -cp target/flink-sql-submit.jar com.github.wuchong.sqlsubmit.SourceGenerator 1000 | $KAFKA_DIR/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic user_behavior
```
代码见https://github.com/cuiwenxu/flink-sql-submit
