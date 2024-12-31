# shell编程

shell 是一个命令行解释器。他接受应用程序/用户命令，然后带哦用操作系统内核。
shell 还是一个功能强大的编程语言，易编写，易调试、灵活性强。

## 1. shell基础

shell脚本以 `#!/bin/bash` 开头，这是用来指明解析器

```shell
# 编写脚本
~]# vim helloworld.sh 
#!/bin/bash

~]#echo "hello world"

# 执行脚本
~]# bash helloworld.sh 
~]# sh helloworld.sh 
# 先设置执行权限，在执行脚本
~]# chmod +x helloworld.sh 
~]# ./helloworld.sh 
```

## 2. shell变量

### 2.2 系统变量

系统变量作用域整个操作系统或者整个用户，临时的用户环境变量指作用当前 bash 以及子 bash，可通过过 set 命令查看当前shell所有变量。

| 系统变量     | 说明     
|----------|--------|
| `$SHEll` | 当前的解释器 |
| `$PWD`   | 当前目录   |
| `$USER`  | 当前用户名  |
| `$HOME`  | 当前用目录  |

### 2.3 自定义变量

shell中通过 `=` 赋值，设置新的变量，通过 `"${}"` 获取变量的指，这里双引号不能替换为单引号，通过unset 撤销变量。 

```shell
~]# A="devops"
~]# echo $A  
devops
~]# echo ${A}
devops
~]# unset A

# 声明静态/只读变量
~]# readonly B=1
~]# unset B  # 静态变量不能通过unset撤销
```

shell 可通过关键字 export 变量提升变量作用域。
```shell
~]# A="linux"
~]# cat 1.sh
#！/bin/bash
echo ${A}

~]# bash 1.sh
~]# export A
~]# bash 1.sh
linux
```

### 2.4 特殊变量

| 特殊变量 | 说明 | 应用场景            |
| --- | --- |-----------------|
| `$n` | n 为数字，`$0` 代表脚本名称，`$1`代表第一个参数,依次类推, 10以上的参数需使用 `${10}` 表示 | 常用于获取脚本名称，和脚本参数 |
| `$#` | 获取所有输入参数个数| |
| `$*` | 代表命令行中所有的参数，`$*`把所有的参数看成一个整体 |
| `$@`| 代表命令行中所有的参数，不过`$@`把每个参数区分对待 |
| `$?` | 最后一次执行的命令的返回状态。如果这个变量的值为0，证明上一个命令正确执
行；如果这个变量的值为非0,则表示上一个命令执行错误 |

```shell
~]# cat test.sh
#!/bin/bash
#
echo $0 $1 $2
echo $#
echo $*
echo $@

echo "====='$*'======"
for i in "$*";do
    echo $i
done
echo "===='$@'====="
for i in "$@";do
    echo $i
done
~]# bash test.sh a b c
test.sh a b
3
a b c
a b c
====='a b c'======
a b c
===='a b c'=====
a
b
c
```

### 2.5 运算符

| 格式         | 案例              | 注意事项       |
|------------|-----------------|------------| 
| `$((运算式))` | ` echo $((2+3))` |
| `$[运算式]`    | `echo $[2+3]`   |
| `expr`     | `expr 2 + 3`    | + 左右必须要有空格 |

## 3. 条件判断

| 基本语法              | 说明       | 判断条件                                                                      |
|-------------------|------------|---------------------------------------------------------------------------|
| `[[ condition ]]` |两个整数之间比较  | `-lt`: 小于,`-le`: 小于等于<br/>`-gt`: 大于, `-ge`: 大于等于<br/>`-ne`: 不等于 |
| `[[ condition ]]` | 按照文件权限  | `-r` 有读的权限（read）<br> `-w` 有写的权限（write）<br>`-x` 有执行的权限（execute）                  |
| `[[ condition ]]` |按照文件类型进行判断 | `-f` 文件存在并且是一个常规的文件（file）<br>`-e` 文件存在（existence）<br>`-d` 文件存在并是一个目录（directory） |

## 4. 逻辑判断

| 符号     | 说明  
|--------|-----|
| `&&`   | 与   |
| `\|\|` | 或 |
| `!`    | 非   |

## 5. 流程控制

### 5.1 if 语句

```shell
#!/bin/bash
#
if [[ ! 18 -lt 15 ]];then
  echo "18 < 15"
fi
```
```shell
#!/bin/bash
#
if [[ ! 18 -gt 15 ]];then
  echo "18 < 15"
elif [[ 18 -gt 15 ]];then
   echo "18 > 15"
fi
```

### 5.2 case 语句

```shell
#!/bin/bash
#
case $1 in
    "A")
        echo "AAAAA"
        ;;
    "A")
        echo "BBBBB"
        ;;
     *)
        echo "other"
        ;;
esac
```

### 5.3 for 语句

```shell
#!/bin/bash
#
#!/bin/bash
#
s=0
#for ((i=0;i<=10;i++));do
#for i in `seq 10`;do
for i in {1..10};do
    s=$[$s+$i]
done
echo $s
```

### 5.4 while 语句

```shell
~]# cat while.sh
#!/bin/bash
#
s=0
i=0
while [[ $i -le 10 ]];do
    s=$[$s+$i]
    i=$[$i+1]
done
echo $s
```

## 6. read读取控制台

```shell
~]# cat read.sh
#!/bin/bash
#
read -t 7 -p "请在7s内输入你的姓名>>>: " NAME
echo $NAME

~]# bash read.sh
请在7s内输入你的姓名>>>: dengyouf
dengyouf
```

## 7. 函数

### 7.1 系统函数

```shell
~]# dirname /root/read.sh
/root
~]# basename /root/read.sh
read.sh
~]# basename read.sh
read.sh
```

### 7.2 自定义函数

函数必须先声明，后使用,返回值可以通过 `$?` 获取,如果不加返回值，则函数的最后一条命令执行结果作为返回值返回

```shell
~]# cat f.sh
#!/bin/bash
#
function sum() {
    s=0
    s=$[ $1 + $2 ]
    echo $s
}


sum $1 $2

~]# bash f.sh 2 3
5
```


