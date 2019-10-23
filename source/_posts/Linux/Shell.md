---
title: Shell
date: 2019-10-18 11:14:21
tags: [Linux,Shell]
categories: Linux	
---

#### Shell简易入门

`shell`是一个用c写的程序，它既是一种命令解释器，同时也是一个程序语言。以下命令可以查看当前Linux装了什么Shell

```bash
echo $SHELL  #查看当前使用的Shell
cat /etc/shells #一共装了哪些Shell
```

#### Execute bash

```bash
1. chmod xxx test.sh
   ./test.sh
2. sh test.sh
3. /bin/sh test.sh
```

#### Shell Variable

##### 定义变量

```bash
name="Jimmy" #变量只能使用英文字母，数字和下划线，首个字符不能以数字开头；不能使用标点符号；不能使用shell关键字
for etcName in `ls /etc` #可以使用语句进行赋值、循环

num=10
readonly num #定义只读变量

unset num #删除变量
```

> 需要注意，变量之后与`=`之间不能有空格，值与`=`也不能有空格：
>
> 比如`num = 10`执行即会报错,需要改为`num=10`

##### 使用变量

```bash
#!/bin/zsh
name="Jimmy"
echo $name #使用时，在变量名前加$
echo ${name} #模板字符串用法

for hobby in Eating Swimming Sleeping; do
    echo "I like ${hobby}"
done;
```

##### Shell 字符串

- 单引号

```bash
str='single dot string' #单引号字符串不支持模板变量，并且不支持单引号转义
```

- 双引号

```bash
name='Jimmy'
str="Hello , my 'name' is ${name}" #双引号支持模板变量，也可以有转义符号
echo $str
```

##### Shell基本变量与操作

```bash
#字符串类
name='Jimmy'

echo ${#name} # 变量名前加`#`输出长度

echo ${name:1:3} # result: imm 截取下标1~3字符串（从0开始）

#数组类
arr=(1 2 3 4)
arr=(
1
2
3
4
)
echo ${arr[0]} #打印第一个元素
echo ${arr[@]} #打印所有元素
echo ${#arr[@]} #打印数组长度 也可使用echo ${#arr[n]}
```

##### Shell运算符

```bash
num=`expr 2 + 2` #shell本身不支持数学运算，可以通过awk、expr实现
num=`expr 2 - 2`#减法
num=`expr 2 \* 2`#乘法
num=`expr 2 / 2`#除法
num=`expr 2 % 2`#余数
num2=$num#赋值
[$num==$num2]#比较两个数字是否相同
[$num!=$num2]#比较两个数字是否不同
```

> expr表达式需要注意：`expr`关键字与后续表达式之间需呀有空格` `,表达式之间也需要有空格`2 + 2`
>
> 布尔条件表达式要放在方括号之间，并且要有空格，例如: `[$a==$b]` 是错误的，必须写成 `[ $a == $b ]`。

##### Shell 简单命令

- echo

```shell
echo hello 
echo "\"hello\""

read test #从标准输入流中读取一行，并赋值
echo "input is ${test}"

echo hello world > aaa.txt #将内容定向至文件
echo 'hello\"' #使用单引号屏蔽转义字符

echo `date` #显示命令执行结果
echo `ls`
```

- printf

``` shell
printf "%-10s %-8s %-4s\n" name gender weight
printf "%-10s %-8s %-4.2f\n" Sally female 49
printf "%-10s %-8s %-4.2f\n" Ford male 61
printf "%-10s %-8s %-4.2f\n" Heisnburg male 66
```

`printf`可以格式化输出，`%s `表示字符,`%f`表示浮点数,`%n`表示整数;`%-10s`:代替符中的数字代表输出一个固定为10的字符

- test

```shell
# 数值测试
num1=20
num2=30
if test ${num1} -eq ${num2}
then
    echo 'The two nums are equals'
else
    echo 'not euquals'
fi
```

`test`命令用于检查条件,在数值测试中`eq`,`ne`,`gt`,`ge`,`lt`,`le`为对比字符，e/eq : equals , n : not , gt : greater than , ge: greater or equals ,lt: less than, le : less or equals 。

此外，test也可以用于文件测试、字符串测试等

 ##### Shell流程

- if

```shell
if con
then
	xxxx
elif
then
	xxx
else
	xxxx
fi
```

- for

```shell
for item in item1 item2 item3 ... itemN
do 
	xxxx
done
```

- while

```shell
while con
do
	xxxx
done
```

- until

```shell
until con
do 
	xxxx
done
```

- case

```shell
case item in 
	xx1) command
	;;
	xx2) command
	;;
	...
	*) default command
	;;
esac
```

##### Shell 函数

```shell
function fun() {
    echo "hello"
	return `expr 10 + 10`
}
fun
echo $? # $?可以获取最近一次的函数返回值

echo "Hello World" | grep -e "World"
echo $? # 值为0，代表匹配到字符串

param(){
		echo "first param : $1"
		echo "second param : $2"
		echo "third param : $3"
		echo "total params : $#"
		echo "All params : $*"
}
param 10 20 30 #在方法名后加入参，即可传参
```

##### Shell I/O重定向

| command             | detail                                  |
| ------------------- | --------------------------------------- |
| `command >/>> file` | 重定向/追加 command命令产生的输出到file |
| `command < file`    | 重定向输入到file                        |

eg:

```shell
touch filename
echo "hello world" > filename
```

