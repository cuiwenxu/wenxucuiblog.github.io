@[TOC](maven-shade-plugin详解)
最近在解决java jar包冲突的时候用到了shade插件，于是从官网仔细看了下shade的详细使用，翻译总结如下，希望能用得到。
>shaded直译为遮盖，这可以比较好的形容maven-shade-plugin插件的功能，比如relocation，打胖jar等

# 介绍
Apache maven shade plugin提供把工程的artifact及其依赖打包到一个uber-jar中并能隐藏起来（比如重命名），shade插件仅仅有一个功能就是创建一个shaded包。
那什么是uber-jar呢，uber在德语中是above或over的意思，在这里表示是从单一的jar提升到“over-jar”，即把所有的依赖都定义到一个jar文件里。
好了，现在我们知道shade插件的基本作用了，现在从官网给出的几个例子看看实际的应用。

# 为uber-jar选择内容
这是官网直译的标题，用我们容易理解的就是通过shade插件我们可以为生成的那个jar包选择包含哪些依赖以及排除哪些依赖。
- 1、支持两种操作include和exclude
- 2、 配置格式：groupId:artifactId[[:type]:classifier]，至少包含groupid和artifactid，type和类名可选
- 3、支持’*’ 和 ‘?’执行通配符匹配
比如，一个示例如下：
```xml
<project>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.0.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <artifactSet>
                <excludes>
                  <exclude>classworlds:classworlds</exclude>
                  <exclude>junit:junit</exclude>
                  <exclude>jmock:*</exclude>
                  <exclude>*:xml-apis</exclude>
                  <exclude>org.apache.maven:lib:tests</exclude>
                  <exclude>log4j:log4j:jar:</exclude>
                </excludes>
              </artifactSet>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ...
</project>
```
# 类重定位(解决jar包冲突)
官网是“Relocating Classes”，如果一个uber-jar会被其他项目引用，uber-jar中依赖的类可能会导致类定位冲突（由于不同版本的jar包引起），`我们可以通过shade插件来将被隐藏的类重定位以使该类只在该uber-jar中使用，这种方式也经常被用来解决jar包冲突问题。`
示例如下：
```xml
<project>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.0.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <relocations>
                <relocation>
                  <pattern>org.codehaus.plexus.util</pattern>
                  <shadedPattern>org.shaded.plexus.util</shadedPattern>
                  <excludes>
                    <exclude>org.codehaus.plexus.util.xml.Xpp3Dom</exclude>
                    <exclude>org.codehaus.plexus.util.xml.pull.*</exclude>
                  </excludes>
                </relocation>
              </relocations>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ...
</project>
```
上述例子是让org.codehaus.plexus.util包和它的子包移动到 org.shaded.plexus.util包中，而类Xpp3Dom和其他的一些则保留在原来的package中。
## 一个解决jar包冲突的例子
代码https://github.com/cuiwenxu/package-test

https://zhuanlan.zhihu.com/p/62796806

# Shaded Artifact附加名字
默认情况下，当执行installed/deployed时候，会默认生成两个jar包，一个以-shaded结尾，这个我们可以配置更改，示例如下：
```xml
<project>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.0.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <shadedArtifactAttached>true</shadedArtifactAttached>
              <shadedClassifierName>myName</shadedClassifierName> <!-- Any name that makes sense -->
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ...
</project>
```
则会生成以-myName结尾的jar包。
# 可执行jar包
要创建一个可执行uber-jar包，也可以将入口添加进来，示例如下：
```xml
<project>
   <groupId>shade.test</groupId>
    <artifactId>shade.test</artifactId>
    <version>1.0-SNAPSHOT</version>
  ...
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.0.0</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
            <configuration>
              <transformers>
                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                  <mainClass>Main</mainClass>
                </transformer>
              </transformers>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
  ...
</project>
```
假设入口函数为：
```java
public class Main {
    public static void main(String[] args) {
        System.out.println("shade Executable test");
    }
}
```
执行mvn package，生成两个jar文件，一个是原始的original-shade.test-1.0-SNAPSHOT.jar，一个是可执行的shade.test-1.0-SNAPSHOT.jar。
执行java -jar shade.test-1.0-SNAPSHOT.jar，效果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210620235315827.png)
以上，是maven shade插件的一些用法。

