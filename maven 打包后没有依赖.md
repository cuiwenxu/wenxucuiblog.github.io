首先明确:

当你使用 Maven 对项目打包时，你需要了解以下 3 个打包 plugin，它们分别是

plugin     | 功能
-------- | -----
maven-jar-plugin	maven | 默认打包插件，用来创建 project jar
maven-shade-plugin | 用来打可执行包，executable(fat) jar
maven-assembly-plugin | 支持定制化打包方式，例如 apache 项目的打包方式
`不管你 dependences 里的 scope 设置为什么, mvn package 出来的 你 src 的 jar 包里, 只会有你的 class 文件, 不会有所依赖的 jar 包, 可以通过 maven assembly 插件来做这个事情.` 但是如果打成 war 包, 是会包含 compile scope 的依赖的. 而 provided 是要容器提供, 比如说 Tomcat, 会到 Tomcat 的 $liferay-tomcat-home\webapps\ROOT\WEB-INF\lib 目录下找.
