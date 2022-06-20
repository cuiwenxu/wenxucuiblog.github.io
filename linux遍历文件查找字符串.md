grep 文本搜索
> grep match_patten file // 默认访问匹配行

常用参数
-o 只输出匹配的文本行 VS -v 只输出没有匹配的文本行
-c 统计文件中包含文本的次数

grep -c "text" filename
-n 打印匹配的行号
-i 搜索时忽略大小写
-l 只打印文件名

在多级目录中对文本递归搜索(程序员搜代码的最爱）：

> grep "class" . -R -n

匹配多个模式

> grep -e "class" -e "vitural" file

限定文件类型
> grep -e "class" -e "vitural" --include="*.lua"  file 
