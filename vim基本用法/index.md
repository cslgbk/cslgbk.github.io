# vim基本用法

<font size=3>原文链接：https://blog.csdn.net/scaleqiao/article/details/45153379</font>
<!--more-->
# 文件命令
```
vim file1 file2     //打开多个文件
:E                  //打开文件
:bn                 //打开下一个文件
:bp                 //打开上一个文件
:e file             //关闭当前文档，打开新文档
:gf                 //打开光标所在字符串的文件名
:x                  //保存并退出
:Sex                //水平分割一个窗口，并浏览文件系统
:Vex                //垂直分割一个窗口，并浏览文件系统
```
# 光标移动
```
gg                  //到文件头部
G                   //到文件尾部
nG                  //到文件第n行
ctrl+f              //下翻一屏
ctrl+d              //上翻一屏
zz                  //将当前行显示在屏幕中央
zt                  //将当前行显示在屏幕顶部
zb                  //将当前行显示在屏幕底部
```
# 剪切复制
```
yy                  //复制当前行
nyy                 //复制当前行以下n行
dd                   //剪切当前行
ndd                 //剪切当前以下n行
P                   //在光标之前粘贴
p                   //在光标之后粘贴
```
# 查找替换
```
/sth                //在后面文本中查找sth
?sth                //在前面文本中查找sth
n                   //向后查找一个
N                   //向前查找一个
:s/old/new          //在当前行将old替换成new
:s/old/new/g        //在整个文本中将old替换成new
:%s/^/xxx/g         //在文本每行行首加入xxx
:%s/$/xxx/g         //在文本每行行末加入xxx
:n1,n2s/old/new/g   //在文本第n1，n2范围进行替换
```

