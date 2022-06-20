>`ps -ef|grep prometheus|awk '{print $2}'|xargs kill -9`

awk '{print $2}'是把进程pid跳出来，xargs命令将标准输入转为命令行参数。
xargs用法举例
>`echo "one two three" | xargs mkdir`

以上命令可以将创建三个文件夹，名字分别为one two three

##### -d指定分隔符
默认情况下，xargs将换行符和空格作为分隔符，把标准输入分解成一个个命令行参数。-d参数可以更改分隔符。
>$ echo -e "a\tb\tc" | xargs -d "\t" echo
a b c

##### -p -t打印执行命令
使用xargs命令以后，由于存在转换参数过程，有时需要确认一下到底执行的是什么命令。

-p参数打印出要执行的命令，询问用户是否要执行。

>$ echo 'one two three' | xargs -p touch
touch one two three ?...

上面的命令执行以后，会打印出最终要执行的命令，让用户确认。用户输入y以后（大小写皆可），才会真正执行。
-t参数则是打印出最终要执行的命令，然后直接执行，不需要用户确认。

>$ echo 'one two three' | xargs -t rm
rm one two three
