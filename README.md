# 基本要求
参考《编译原理及实践》的TINY语言编译器（已上传到群中）完成TINY+语言（见附录A）的解释器：即给定满足TINY+语言的源代码输入，你的解释器可以给出对其的解释执行结果。

# 完成流程
## 1. 文法描述
经对比，TINY+语法与TINY语法区别仅为标黄部分
program 				declaration-list; stmt-sequence

declaration-list 	→ 	declaration-list declaration | declaration

declaration 		→ 	type-specifier identifier; 

type-specifier 		→ 	int | char

stmt-sequence 	→	stmt-sequence; statement | statement

statement			→	if-stmt | repeat-stmt | assign-stmt | read-stmt | write-stmt

if-stmt 		→	  	if (exp) then stmt-sequence end
                | if (exp) then stmt-sequence else stmt-sequence end
                
repeat-stmt 	→		repeat stmt-sequence until exp

assign-stmt 		→	identifier := exp

read-stmt 	→	 	read identifier

write-stmt 	→		write exp

exp 		→			simple-exp comparson-op simple-exp | simple-exp

comparison 	→	 	< | =

simple-exp		→	simple-exp addop term | term

addop		→	 	+ | -

term 		→	 	term mulop factor | factor

mulop 	→		 	* | /

factor 		→		 (exp) | number | identifier

number		→		(+|-)?[1-9][0-9]*

identifier	→		 	[a-zA-Z]([0-9]| [a-zA-Z])*



DFA如图所示，当前记号（token）属于INID时，需要检测其是否为关键字。相对于TINY，TINY+的关键字种类增加了CHAR与INT。因此不用更改原TINY语言的DFA

## 2.设计与实现

### 2．1扫描处理scan.c
相对于TINY语言原有scan.c程序，TINY+的scan.c程序需要

1. 增加reservedWords[]即保留字中的种类，添加INT与CHAR。
2. 将MAXRESERVED（保留字个数）从8增加至10。
扫描过程的整体流程不变。在当前记号类型（TokenType）为ID时，需要检测其是否属于关键字。 

### 2．2 上下无关文法及分析 构造抽象语法树 parse.c
#### 2．2．1针对新增文法declaration-list：
program 		→		declaration-list; stmt-sequence

declaration-list 	→ 	declaration-list declaration | declaration

declaration 		→ 	type-specifier identifier; 

type-specifier 		→ 	int | char

采用与TINY语言源程序相同的递归下降分析器，左结合方式。

因此将其替换为

program 				declaration-list; stmt-sequence

declaration-list 	→ 	declaration declaration-list’

declaration-list’   →  declaration declaration-list’

declaration 		→ 	type-specifier identifier; 

type-specifier 		→ 	int | char

同时，增加program函数用以衔接declaration-list与 stmt-sequence

#### 2.2.2 对于语法if-stmt
if-stmt 		→  	if (exp) then stmt-sequence end

                | if (exp) then stmt-sequence else stmt-sequence end
                
从原来的exp变为(exp)，增加对左括号和有括号的match操作。

#### 2.2.3 总述
原TINY语言共将树节点分为两种类型，StmtKind和ExpKind。其中StmtKind对应
if-stmt 		→	  	if (exp) then stmt-sequence end

                | if (exp) then stmt-sequence else stmt-sequence end
                
repeat-stmt 		→	repeat stmt-sequence until exp

assign-stmt 	→		identifier := exp

read-stmt 	→	 	read identifier

write-stmt 		→	write exp

五种规则的左表达式。ExpKind对应其余所有表达式。

在TINY+语言中，因为新增文法declaration-list，增加了一种新的节点类型DclKind。该节点内设两种类型IntK,与CharK。每一个declaration对应一个DclKind节点。该节点的兄弟节点为下一表达式所对应节点（declaration或statement）。

### 2．3语义分析 analyze.c
#### 2.3.1 checkNode 函数
在TINY语言原程序中，对于checkNode的遍历顺序，若当前节点有子节点，则先遍历子节点。否则先遍历当前节点，再遍历当前节点的兄弟节点。TINY+语言在保持原遍历顺序不变的情况下，加入一个针对DclK节点与IdK节点的验证。

由于新语法规则的引入，现在TINY+语言中的变量均需要事先声明才可以使用。在遍历语法树Dcl节点时，记录下变量名与对应的变量类型（int或char）于新建立的VariableInt[] 和VariableChar[]表中。
在当遇到节点类型为ExpK中的IdK，即该节点表示标识符时，要查找该标识符是否被定义。调用新增的lookforExist函数，遍历VariableInt[] 和VariableChar[]表，若找到，则确定该节点的type，并返回1。否则返回0。

当返回节点为0时，该标识符存在未定义的情况，报错
