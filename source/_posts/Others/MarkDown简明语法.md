---
title: MarkDown简明语法
date: 2018-08-14 16:30:54
tags: [MarkDown,语言]
categories: 工具

---
 MarkDown几种常用的语法
<!-- more -->
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
```
# 一级标题
## 二级标题
### 三级标题
#### 四级标题
##### 五级标题
```

***
### 无序列表
* 内容
* 内容
* 内容

```
* 内容
* 内容
* 内容
```
***

### 有序列表
1.内容
2.内容
3.内容
```
1.内容
2.内容
3.内容
```

***
### 区块引用
> 引用文字

```
> 引用
```

***
```
*** 下划线
```
### 链接

[百度](http://www.baidu.com)一下，你就知道
```
[内容](连接)
eg: [百度](http://www.baidu.com)
```

***
![图片](https://www.baidu.com/img/bd_logo1.png)
```
![图片名称](连接)
eg: [百度](https://www.baidu.com/img/bd_logo1.png)
```

***
### 表格

表头1 | 表头2 
---|---
行1列1 | 行1列2
行2列1 | 行2列2

```
header 1 | header 2
---|---
row 1 col 1 | row 1 col 2
row 2 col 1 | row 2 col 2
```
***
### 代码
如何学习`Angular2`?
```java
public static void test(){
	String a = "Hello";
	String b = "world";
	System.out.println(a+" "+b);
}
```

```
如何学习`Angular2`?
```java
public static void test(){
	String a = "Hello";
	String b = "world";
	System.out.println(a+" "+b);
}
```

---

### 强调
*倾斜字体*
_倾斜字体_
**加粗字体**
__加粗字体__

```
*倾斜字体*
_倾斜字体_
**加粗字体**
__加粗字体__
```

---

### 转义字符

* \\
* \`
* \~
* \*
* \_
* \-
* \+
* \.
* \!

```
* \\
* \`
* \~
* \*
* \_
* \-
* \+
* \.
* \!
```
---
### 删除线
~~删除文本~~
```
~~删除文本~~
```
---
### 待办事项以及完成事项
- [ ] - TODO
- [x] - HAVA DONE
```
- [ ] - TODO
- [x] - HAVA DONE
```
---
### 换行
如果另起一行，只需在当前行结尾加 2 个空格  
这行就会新起一行
```
在当前行的结尾加 2 个空格  
这行就会新起一行
```