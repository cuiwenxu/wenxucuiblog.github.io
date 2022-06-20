- 字符串通过ByteBuffer->FileChannel->文件 demo
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021031713241141.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
```java
import java.io.FileOutputStream;
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class StringWriteFile {


    public static void main(String[] args) throws IOException {
        String str = "hello world";
        //创建一个流，并获取其channel
        FileOutputStream fileOutputStream = new FileOutputStream("nio_output");
        FileChannel fileChannel = fileOutputStream.getChannel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        byteBuffer.put(str.getBytes());
        byteBuffer.flip();
        //将bytebuffer写入channel
        fileChannel.write(byteBuffer);
        fileOutputStream.close();
    }


}
```
