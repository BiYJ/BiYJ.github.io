---
title: shell
---

[shell](https://www.runoob.com/linux/linux-shell.html)

`#!` 是一个约定的标记，它告诉系统这个脚本需要什么解释器来执行，即使用哪一种 Shell。

运行 Shell 脚本有两种方法：

1. 作为可执行程序

	```
	chmod +x ./test.sh  # 使脚本具有执行权限
	./test.sh  # 执行脚本
	```

2. 作为解释器参数

	这种运行方式是，直接运行解释器，其参数就是 shell 脚本的文件名，如：
	
	```
	/bin/sh test.sh
	/bin/php test.php
	```
	
	这种方式运行的脚本，不需要在第一行指定解释器信息，写了也没用。
	
```
#!/bin/bash

# 向窗口输出文本
echo "Hello word!"


###----------------
#   变量
###-----------------

# 定义变量。变量名和等号之间不能有空格
name="Jack Ma"
# 使用变量
echo $name
# 花括号帮助解释器识别变量的边界，避免 good 被识别为变量名的一部分
echo "${name}good!"

#只读变量
readonly weburl="www.baidu.com"
echo $weburl
#不能修改只读变量
weburl="www.google.com"

# 删除变量。变量被删除后不能再次使用。unset 命令不能删除只读变量。
unset name
echo $names


###----------------
#   字符串
###-----------------

# 单引号字符串。单引号里的任何字符都会原样输出，单引号字符串中的变量是无效的；单引号字串中不能出现单独一个的单引号（对单引号使用转义符后也不行），但可成对出现，作为字符串拼接使用。
str1='a string'
# 双引号字符串。双引号里可以有变量；双引号里可以出现转义字符
str2="a string with \"$str1\""
echo -e $str2
# 获取字符串长度
ehco ${#str2}
# 获取子字符串
echo ${str2:1:4}
# 查找字符 i 或 w 的位置（哪个字母先出现就计算哪个）
echo `expr index "$str2" iw`


###----------------
#   数组
###-----------------

# 用括号来表示数组，数组元素用"空格"符号分割开
arr1=("1" "2" 3)
arr2=(
"4"
"5555555"
"6"
)
# 单独定义数组的某个元素
arr2[2]="7"
# 读取数组元素值
echo ${arr[2]}
# 获取数组中的所有元素
echo ${arr[@]}
# 获取数组元素的个数
ehco ${#arr[@]}
ehco ${#arr[*]}
# 获取数组单个元素的长度
echo ${#arr2[1]}


###----------------
#   注释
###-----------------

:<<EOF
注释内容...
注释内容...
注释内容...
EOF

:<<!
注释内容...
注释内容...
注释内容...
!

###----------------
#   参数传递
###-----------------

#  默认 $0 为执行的文件名
echo "文件名：$0"
echo "传递参数一：$1"
echo "传递参数二：$2"
echo "传递参数三：$3"
# 传递到脚本的参数个数
echo "参数个数：$#"
# 以一个单字符串显示所有向脚本传递的参数，不包含 $0
echo "参数内容：$*"
# 脚本运行的当前进程ID号
echo "进程 ID：$$"
# 后台运行的最后一个进程的ID号
echo "进程 ID：$!"
# $* 和 $@ 的区别
for s in $*; do
    echo $s
done

for s in $@; do
    echo $s
done


###----------------
#   算术运算符
###-----------------

# expr 是一款表达式计算工具，使用它能完成表达式的求值操作
# 表达式和运算符之间要有空格，2+2 是不对的，必须写成 2 + 2
val=`expr 2 + 2`
echo $val

a=10
b=20
c=10
val=`expr $a + $b`
echo "a + b : $val"

val=`expr $a - $b`
echo "a - b : $val"

# 乘号（*）前边必须加反斜杠（\）才能实现乘法运算
val=`expr $a \* $b`
echo "a * b : $val"

val=`expr $b / $a`
echo "b / a : $val"

val=`expr $b % $a`
echo "b % a : $val"

# 条件表达式要放在方括号之间，并且要有空格，[$a==$b] 是错误的，必须写成 [ $a == $b ]
# if...then...fi 是条件语句
# 写在一行使用分号分隔
if [ $a == $b ]; then echo "a 等于 b"; fi

if [ $a == $c ]
then
    echo "a 等于 c"
fi

if [ $a != $b ]
then
    echo "a 不等于 b"
fi


###----------------
#   关系运算符
###-----------------

# -eq：检测两个数是否相等
if [ $a -eq $b ]
then
    echo "$a -eq $b : a 等于 b"
else
    echo "$a -eq $b: a 不等于 b"
fi

# -ne：检测两个数是否不相等
if [ $a -ne $b ]
then
    echo "$a -ne $b: a 不等于 b"
else
    echo "$a -ne $b : a 等于 b"
fi

# -gt：greater than，检测左边的数是否大于右边的
if [ $a -gt $b ]
then
    echo "$a -gt $b: a 大于 b"
else
    echo "$a -gt $b: a 不大于 b"
fi

# -lt：less than，检测左边的数是否小于右边的
if [ $a -lt $b ]
then
    echo "$a -lt $b: a 小于 b"
else
    echo "$a -lt $b: a 不小于 b"
fi

# -ge：检测左边的数是否大于等于右边的
if [ $a -ge $b ]
then
    echo "$a -ge $b: a 大于或等于 b"
else
    echo "$a -ge $b: a 小于 b"
fi

# -le：检测左边的数是否小于等于右边的
if [ $a -le $b ]
then
    echo "$a -le $b: a 小于或等于 b"
else
    echo "$a -le $b: a 大于 b"
fi


###----------------
#   布尔运算符
###-----------------

# !：非运算
if [ $a != $b ]
then
    echo "$a != $b : a 不等于 b"
else
    echo "$a == $b : a 等于 b"
fi

# -a：AND，与运算
if [ $a -lt 100 -a $b -gt 15 ]
then
    echo "$a 小于 100 且 $b 大于 15 : 返回 true"
else
    echo "$a 小于 100 且 $b 大于 15 : 返回 false"
fi

# -o：OR，或运算
if [ $a -lt 5 -o $b -gt 100 ]
then
    echo "$a 小于 5 或 $b 大于 100 : 返回 true"
else
    echo "$a 小于 5 或 $b 大于 100 : 返回 false"
fi


###----------------
#   逻辑运算符
###-----------------

# &&：逻辑的 AND
if [[ $a -lt 100 && $b -gt 100 ]]
then
    echo "返回 true"
else
    echo "返回 false"
fi

# ||：逻辑的 OR
if [[ $a -lt 100 || $b -gt 100 ]]
then
    echo "返回 true"
else
    echo "返回 false"
fi


###----------------
#   字符串运算符
###-----------------

string1="abc"
string2="efg"

# =：检测两个字符串是否相等
if [ $string1 = $string2 ]
then
    echo "$a = $b : a 等于 b"
else
    echo "$a = $b: a 不等于 b"
fi

# !=：检测两个字符串是否不等，与 = 相反
if [ $a != $b ]
then
    echo "$a != $b : a 不等于 b"
else
    echo "$a != $b: a 等于 b"
fi

# -z：检测字符串长度是否为 0
if [ -z $a ]
then
    echo "-z $a : 字符串长度为 0"
else
    echo "-z $a : 字符串长度不为 0"
fi

# -n：检测字符串长度是否大于 0，与 -z 相反
if [ -n "$a" ]
then
    echo "-n $a : 字符串长度不为 0"
else
    echo "-n $a : 字符串长度为 0"
fi

# $：检测字符串是否为空
if [ $a ]
then
    echo "$a : 字符串不为空"
else
    echo "$a : 字符串为空"
fi


###----------------
#   文件测试运算符
###-----------------

file="/var/www/runoob/test.sh"

# -r：read
if [ -r $file ]
then
    echo "文件可读"
else
    echo "文件不可读"
fi

# -w：write
if [ -w $file ]
then
    echo "文件可写"
else
    echo "文件不可写"
fi

# -x：exe
if [ -x $file ]
then
    echo "文件可执行"
else
    echo "文件不可执行"
fi

if [ -f $file ]
then
    echo "文件为普通文件"
else
    echo "文件为特殊文件"
fi

# -d：
if [ -d $file ]
then
    echo "文件是个目录"
else
    echo "文件不是个目录"
fi

if [ -s $file ]
then
    echo "文件不为空"
else
    echo "文件为空"
fi

# -e：exist
if [ -e $file ]
then
    echo "文件存在"
else
    echo "文件不存在"
fi


###----------------
#   echo
###-----------------

# 双引号可以省略
echo "It is a test"
echo It is a test

# read 命令从标准输入中读取一行，并把输入行的每个字段的值指定给  shell 变量
read name
echo name is $name

### 注意下面三种的区别
echo "OK!"
echo "It is a test"
# -e 开启转义
echo -e "OK! \n"
echo "It is a test"
# \c 不换行
echo -e "OK! \c"
echo "It is a test"

# 显示结果定向至文件
echo "It is a test" > myfile

# 原样输出字符串，不进行转义或取变量（用单引号）
echo "$name"
echo '$name\"'

# 显示命令执行结果
echo `date`


###----------------
#   printf：printf 由 POSIX 标准所定义，因此使用 printf 的脚本比使用 echo 移植性好
#                 默认 printf 不会像 echo 自动添加换行符，我们可以手动添加 \n
###-----------------

# %-10s 指一个宽度为 10 个字符（-表示左对齐，没有则表示右对齐）
printf "%-10s %-8s %-4s\n"   姓名 性别 体重kg
# %-4.2f 指格式化为小数，其中.2 指保留 2 位小数
printf "%-10s %-8s %-4.2f\n" 郭靖 男 66.1234
printf "%-10s %-8s %-4.2f\n" 杨过 男 48.6543
printf "%-10s %-8s %-4.2f\n" 郭芙 女 47.9876

# format-string为双引号
printf "%d %s\n" 1 "abc"
# 单引号与双引号效果一样
printf '%d %s\n' 1 "abc"
# 没有引号也可以输出
printf %s abcdef
# 格式只指定了一个参数，但多出的参数仍然会按照该格式输出，format-string 被重用
printf %s abc def

printf "%s\n" abc def

printf "%s %s %s\n" a b c d e f g h i j

# 如果没有 arguments，那么 %s 用NULL代替，%d 用 0 代替
printf "%s and %d \n"


###----------------
#   test 命令
###-----------------

num1=100
num2=100
# -eq：等于则为真；
# -ne：不等于则为真；
# -gt：大于则为真；
# -ge：大于等于则为真；
# -lt：小于则为真；
# -le：小于等于则为真
if test $[num1] -eq $[num2]
then
    echo '两个数相等！'
else
    echo '两个数不相等！'
fi
# 代码中的 [] 执行基本的算数运算
a=5
b=6
result=$[a+b] # 注意等号两边不能有空格

s1="ru1noob"
s2="runoob"
# =：等于则为真；
# !=：不相等则为真
# -z 字符串：字符串的长度为零则为真
# -n 字符串：字符串的长度不为零则为真
if test $s1 = $s2
then
    echo '两个字符串相等!'
else
    echo '两个字符串不相等!'
fi

# -e 文件名    如果文件存在则为真
# -r 文件名    如果文件存在且可读则为真
# -w 文件名    如果文件存在且可写则为真
# -x 文件名    如果文件存在且可执行则为真
# -s 文件名    如果文件存在且至少有一个字符则为真
# -d 文件名    如果文件存在且为目录则为真
# -f 文件名    如果文件存在且为普通文件则为真
# -c 文件名    如果文件存在且为字符型特殊文件则为真
# -b 文件名    如果文件存在且为块特殊文件则为真
cd /bin
if test -e ./bash
then
    echo '文件已存在!'
else
    echo '文件不存在!'
fi

###----------------
#   流程控制
###-----------------

# if - then - elif - then - else - fi
a=10
b=20
if [ $a == $b ]
then
    echo "a 等于 b"
elif [ $a -gt $b ]
then
    echo "a 大于 b"
elif [ $a -lt $b ]
then
    echo "a 小于 b"
else
    echo "没有符合的条件"
fi

# for 循环
for loop in 1 2 3 4 5
do
echo "The value is: $loop"
done

# while 循环
int=1
while(( $int<=5 ))
do
    echo $int
    let "int++"
done

# 无限循环
while :
do

done

while true
do

done
```