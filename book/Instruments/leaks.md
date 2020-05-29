### 内存检查

使用工具leak 检测内存泄漏

##### 首先

在打包的时候 在Xcode中设置Debug information Format：

> Building Setting -> Debug Information Format -> Debug -> DWARF with dSYM File

只有设置为DWARF with dSYM File 在使用leak内存检测时检测工具才能解析定位到的代码，不然就没办法查看；

##### 内存检测

1、打开工具instrument 选择 leak;

2、运行工具

##### 分析

出现红叉，表示有内存泄漏，这个时候点击暂停，然后选择 leaks-> Call Tree, 然后使用鼠标选中红叉部分，这个时候就能看到内存泄漏部分定位，然后点开可以查看具体代码位置（如果不设置Debug information Format ，是没办法解析的）

