---
layout:     post
title:      "了解Markdown语法"
date:       2018-06-11
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - 语法
---

<font id="last-updated">最后更新于：2018-06-30</font>


blog的文章都用markdown格式，所以需要了解语法结构，方便以后写博客。

> Markdown 是一种用来写作的轻量级「标记语言」，它用简洁的语法代替排版。

使用 Markdown 的优点：
- 专注你的文字内容而不是排版样式，安心写作。
- 轻松的导出 HTML、PDF 和本身的 .md 文件。
- 纯文本内容，兼容所有的文本编辑器与字处理软件。
- 随时修改你的文章版本，不必像字处理软件生成若干文件版本导致混乱。
- 可读、直观、学习成本低。

Markdown编辑器在Windows下试用了几个都觉得不好，可以使用web在线编辑器。
如[作业部落](https://www.zybuluo.com/mdeditor)、[马克飞象](https://maxiang.io/)、[简书](https://www.jianshu.com/)

[TOC]


# 语法简要规则

## 1.标题
```
# 一级标题

## 二级标题

### 三级标题
```
使用#以此类推，并且建议在#后加一个空格，总共有6级标题

也可以通过下划线表示一级和二级标题
```
这是一个一级标题
============================

这是一个二级标题
--------------------------------------------------

### 这是一个三级标题
```
也可以选择在行首加井号表示不同级别的标题 (H1-H6)，例如：# H1, ## H2, ### H3，#### H4


## 2.列表
在文字前加上 *，+，- 表示无序列表；  
在文字前加上1. 2. 3.表示有序列表。

1. first
2. second
3. third

使用 - 与 [ ]或 [X] 表示打钩与否的列表

- [ ] 支持以 PDF 格式导出文稿
- [ ] 改进 Cmd 渲染算法，使用局部渲染技术提高渲染效率
- [x] 新增 Todo 列表功能
- [x] 修复 LaTex 公式渲染问题
- [x] 新增 LaTex 公式编号功能
- [ ] **Cmd Markdown 开发**
    - [ ] 改进 Cmd 渲染算法，使用局部渲染技术提高渲染效率
    - [ ] 支持以 PDF 格式导出文稿
    - [x] 新增Todo列表功能


## 3.引用
使用>表示
> 这里是引用

## 4.图片与链接
图片为：\!\[描述](链接地址);

链接为：\[描述](链接地址 <title="Title">)。

[跳到我的首页](https://zhouj000.github.io/ "myHome")

参考式的链接是在链接文字的括号后面再接上另一个方括号，而在第二个方括号里面要填入用以辨识链接的标记：  
`使用This is [an example] [id] reference-style link.`

This is [an example][id] reference-style link.

接着，在文件的任意处，你可以把这个标记的链接内容定义出来：  
`[id]: https://zhouj000.github.io/  "Optional Title Here"`

[id]: https://zhouj000.github.io/  "Optional Title Here"

隐式链接标记功能让你可以省略指定链接标记，这种情形下，链接标记会视为等同于链接文字，要用隐式链接标记只要在链接文字后面加上一个空的方括号
```
[Google][]

[Google]: http://google.com/
```

自动链接
`<https://zhouj000.github.io/>`

<https://zhouj000.github.io/>

## 5.代码框

只需要用1个 ` 把中间的代码包裹起来，表示行内代码块。

` public static void main(String[] args) {	System.out.println("hello world!");	}`

使用3个 ` 跨行代码包裹起来，表示代码块。
``` 
//
//                       _oo0oo_
//                      o8888888o
//                      88" . "88
//                      (| -_- |)
//                      0\  =  /0
//                    ___/`---'\___
//                  .' \\|     |// '.
//                 / \\|||  :  |||// \
//                / _||||| -:- |||||- \
//               |   | \\\  -  /// |   |
//               | \_|  ''\---/''  |_/ |
//               \  .-\__  '-'  ___/-. /
//             ___'. .'  /--.--\  `. .'___
//          ."" '<  `.___\_<|>_/___.' >' "".
//         | | :  `- \`.;`\ _ /`;.`/ - ` : | |
//         \  \ `_.   \_ __\ /__ _/   .-` /  /
//     =====`-.____`.___ \_____/___.-`___.-'=====
//                       `=---='
//
//
//     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
//
//               佛祖保佑         永无BUG
//
```

在3个`后面加上语言，例如python，高亮一段代码
```python
@requires_authorization
class SomeClass:
    pass

if __name__ == '__main__':
    # A comment
    print 'hello world'
```


## 6.分割线与删除线
只需要3个 * 号

***

这里是分割线下方

使用 ~~ 表示删除线。

~~这是一段错误的文本。~~



## 7.粗体与斜体

使用2个 *(或_) 包含的是粗体，使用1个 *(或_) 包含的是斜体:

**这里是粗体**  *这里是斜体*

## 8.特殊标记与换行

使用 [^keyword] 表示注脚，并在最后同样的标记后写上标注
 
书写一个质能守恒公式[^LaTeX]


单个回车 视为空格。

连续回车

才能分段。

行尾加两个空格，这里->  
即可段内换行。

常用需转义字符：
<table>
    <tr>
        <th>显示结果</th>
        <th>描述</th>
        <th>实体名称</th>
		<th>实体编号</th>
    </tr>
    <tr>
        <td> </td>
        <td>空格</td>
        <td>&nbsp;</td>
		<td>&#160;</td>
    </tr>
	<tr>
        <td><</td>
        <td>小于号</td>
        <td>&lt;</td>
		<td>&#60;</td>
    </tr>
	<tr>
        <td>></td>
        <td>大于号</td>
        <td>&gt;</td>
		<td>&#62;</td>
    </tr>
	<tr>
        <td>&</td>
        <td>和号</td>
        <td>&amp;</td>
		<td>&#38;</td>
    </tr>
	<tr>
        <td>“</td>
        <td>引号</td>
        <td>&quot;</td>
		<td>&#34;</td>
    </tr>
	<tr>
        <td>‘</td>
        <td>撇号</td>
        <td>&apos;(IE不支持)</td>
		<td>&#39;</td>
    </tr>
</table>



## 9.表格

比较麻烦的方法:
```
| Item      |    Value | Qty  |
| :-------- | --------:| :--: |
| Computer  | 1600 USD |  5   |
| Phone     |   12 USD |  12  |
| Pipe      |    1 USD | 234  |
```

| Item      |    Value | Qty  |
| :-------- | --------:| :--: |
| Computer  | 1600 USD |  5   |
| Phone     |   12 USD |  12  |
| Pipe      |    1 USD | 234  |


还是直接使用HTML标签<table>来实现

<table>
    <tr>
        <th rowspan="2">值班人员</th>
        <th>星期一</th>
        <th>星期二</th>
        <th>星期三</th>
    </tr>
    <tr>
        <td>李强</td>
        <td>张明</td>
        <td>王平</td>
    </tr>
</table>


## 10.字体与颜色

使用html标签：

`<font face="黑体">我是黑体字</font>`
<font face="黑体">我是黑体字</font>

`<font color=#0099ff size=4 face="黑体">海蓝，黑体，size为4</font>`
<font color="#0099ff" size="4" face="黑体">海蓝，黑体，size为4</font> 

参考： [颜色代码表](http://www.w3school.com.cn/tags/html_ref_colornames.asp)


***

[^LaTeX]: 支持 **LaTeX** 编辑显示支持，例如：$\sum_{i=1}^n a_i=0$