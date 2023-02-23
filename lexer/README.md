编译原理实验终于迎来了实验一，本实验要求实现一个词法分析器。回忆理论课学习的词法分析过程，词法分析要求把一个预处理过的文件里面的字符序列，转换成一个个具有特定含义的 Token 序列，即终结符序列。由于我们的编译器实现是对标 Clang，所以我们先以  Clang 为例，看看它在词法分析阶段做了些什么。

## Clang 的词法分析

以下面这段 C 代码（`tester/functional/000_main.sysu.c`）为例：

```C
int main(){
    return 3;
}
```

在命令行下输入：

```bash
( export PATH=$HOME/sysu/bin:$PATH \
  CPATH=$HOME/sysu/include:$CPATH \
  LIBRARY_PATH=$HOME/sysu/lib:$LIBRARY_PATH \
  LD_LIBRARY_PATH=$HOME/sysu/lib:$LD_LIBRARY_PATH &&
  clang -E tester/functional/000_main.sysu.c |
  clang -cc1 -dump-tokens 2>&1 )
```

将输出

```bash
int 'int'        [StartOfLine]  Loc=<tester/functional/000_main.sysu.c:1:1>
identifier 'main'        [LeadingSpace] Loc=<tester/functional/000_main.sysu.c:1:5>
l_paren '('             Loc=<tester/functional/000_main.sysu.c:1:9>
r_paren ')'             Loc=<tester/functional/000_main.sysu.c:1:10>
l_brace '{'             Loc=<tester/functional/000_main.sysu.c:1:11>
return 'return'  [StartOfLine] [LeadingSpace]   Loc=<tester/functional/000_main.sysu.c:2:5>
numeric_constant '3'     [LeadingSpace] Loc=<tester/functional/000_main.sysu.c:2:12>
semi ';'                Loc=<tester/functional/000_main.sysu.c:2:13>
r_brace '}'      [StartOfLine]  Loc=<tester/functional/000_main.sysu.c:3:1>
eof ''          Loc=<tester/functional/000_main.sysu.c:3:2>
```

可以发现，上面的 `clang -cc1 -dump-tokens 2>&1` 即进行词法分析后，输出的每一行包含：

1. （一个）**有效 token**。
2. （一个）由单引号包裹的匹配串。
3. （若干）识别上一个有效 token 到这个有效 token 间遇到的无效 token。
4. （一个）有效 token 的位置。

## 实验要求

你被希望完成一个词法分析器 `sysu-lexer`，产生与 `clang -cc1 -dump-tokens 2>&1` **相当的内容**。预期的代码行数为 250 行，预期的完成时间为 2 小时 ～ 6 小时。

`lexer/lexer.l` 提供了一个基于 flex 实现的模板，你可以基于此继续实现完整的逻辑，也可以使用其他的工具实现，如 `antlr4`，但不得使用其提供的 [C 语言模板](https://github.com/antlr/grammars-v4/blob/master/c/C.g4)；也不得使用任何封装好的库直接获得 token，如 `libclang`。

### flex快速词法分析生成器简介
flex是一个快速词法分析生成器（与网页设计领域中的flex正好同名），它可以将用户用正则表达式写的分词匹配模式构造成一个有限状态自动机（一个C函数），目前很多编译器都采用它来生成词法分析器。如果你暂时不清楚正则表达式和有限状态自动机的概念，可以参考[此链接](https://pandolia.net/tinyc/ch7_lexical_basic.html)中7.3和7.4相关内容。如果下一节中的flex模版你无法很好地使用，你也可以打开[此链接](https://pandolia.net/tinyc/ch8_flex.html)中8.1相关内容进行学习。

### *基于 flex 实现

`lexer/lexer.l` 提供了一个基于 flex 实现的模板，它的部分代码如下：

```c
%{
#include <cctype>
#include <cstdio>
#include <string>
#define YYEOF 0
int yylex();
int main() {
  do {
  } while (yylex() != YYEOF);
}
std::string yyloc = "<stdin>";
int yyrow = 1, yycolumn = 1, yycolpre = 1;
#define YY_USER_ACTION                                                         \
  do {                                                                         \
    yycolumn += yyleng;                                                        \
  } while (0);
%}
%option noyywrap  
%%
...
return {
  std::fprintf(yyout, "return '%s'\t\tLoc=<%s:%d:%d>\n", yytext, yyloc.c_str(),
               yyrow, yycolumn - yyleng);
  return ~YYEOF;
}
...
%%
```

程序由 2 或 3 段组成，分别是：

```
/* P1: declarations(定义段) */
%{
  
%}

%%
  /* P2: translation rules(规则段) */
%%

/* P3: auxiliary functions(用户辅助程序段，c函数), 注意模板中没有定义辅助函数*/
```

- 第一段（定义段）`%{`  和 `%}` 之间的部分，由C语言编写的，包括头文件include、宏定义、全局变量定义、函数声明等；

- 第二段（**规则段**）`%%...%%` 之间部分，为一系列**匹配模式**(**正则表达式**)和动作(**C代码**)。当 flex 扫描程序运行时，它把输入和规则段的正则表达式进行匹配，每次发现一个匹配就执行该模式后定义的相关 C 代码。

  > 模式具有二义性时，即相同的输入可能会被不同的模式匹配。flex 使用两个简单的规则：
  >
  > -  贪婪匹配：匹配输入时匹配尽可能长的字符串。
  > -  若两个模式都可以匹配，则匹配在程序中更早的出现的模式。

flex 更加详细的学习参考 [Lexical Analysis With Flex](http://westes.github.io/flex/manual/) 。

下面演示目前模板生成的 `sysu-lexer`，在命令行下输入

```bash
# 进入到 SYsU-lang 目录，路径可能需要根据自己的实际情况调整
cd SYsU-lang

# 编译安装
# `${CMAKE_C_COMPILER}` 仅用于编译 `.sysu.c`
# 非 SYsU 语言的代码都将直接/间接使用 `${CMAKE_CXX_COMPILER}` 编译（后缀为 `.cc`）
rm -rf $HOME/sysu
cmake -G Ninja \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_INSTALL_PREFIX=$HOME/sysu \
  -DCMAKE_MODULE_PATH=$(llvm-config --cmakedir) \
  -DCPACK_SOURCE_IGNORE_FILES=".git/;tester/third_party/" \
  -B $HOME/sysu/build
cmake --build $HOME/sysu/build
cmake --build $HOME/sysu/build -t install
```

然后进入 `$HOME/sysu/bin` 目录：

```bash
$ cd $HOME/sysu/bin && ls
sysu-compiler  sysu-generator  sysu-lexer  sysu-linker  sysu-optimizer  sysu-parser  sysu-preprocessor  sysu-translator
```

`sysu-lexer` 即为模板生成的词法分析器，它具备初步的词法分析功能，但是**功能不完整，需要自行补全**。

一个例子演示如何使用 `sysu-lexer`:

```bash
# 进入到 SYsU-lang 目录，路径可能需要根据自己的实际情况调整
$ cd SYsU-lang

# 目前仅支持从标准输入读入，所以使用了 '<' 重定向输入
$ $HOME/sysu/bin/sysu-lexer < tester/functional/000_main.sysu.c
int 'int'               Loc=<<stdin>:1:1>
identifier 'main'               Loc=<<stdin>:1:5>
l_paren '('             Loc=<<stdin>:1:9>
r_paren ')'             Loc=<<stdin>:1:10>
l_brace '{'             Loc=<<stdin>:1:11>
return 'return'         Loc=<<stdin>:2:5>
numeric_constant '3'            Loc=<<stdin>:2:12>
semi ';'                Loc=<<stdin>:2:13>
r_brace '}'             Loc=<<stdin>:3:1>
eof ''          Loc=<<stdin>:2:14>
```

可以看到模板的 `sysu-lexer` 产生了和 Clang 一致的词法分析结果。

### 有效 token

词法分析要求识别有效的 token，那么哪些是有效合法的 token 呢？本编译原理实验是要求大家实现一个 SYsU 教学语言，我们在前一节给出了该语言终结符(token)的特点，具体在[下表](https://github.com/arcsysu/SYsU-lang/discussions/10)中进行总结：

```
token       name

const       const
,           comma
;           semi
int         int
[           l_square
]           r_square
=           equal
{           l_brace
}           r_brace
ident       identifier
(           l_paren
)           r_paren
void        void
if          if
else        else
while       while
break       break
continue    continue
return      return
intconst    numeric_constant
+           plus
-           minus
!           exclaim
*           star
/           slash
%           percent
<           less
>           greater
<=          lessequal
>=          greaterequal
==          equalequal
!=          exclaimequal
&&          ampamp
||          pipepipe
string      string_literal
char        char
```

## 评分规则

本实验的评分分为两部分：基础部分和挑战部分。

- 对于基础部分的实验，由低到高分别给出三档实验要求，并要求通过对应的自动评测。详见自动评测细则一节。
- 对于挑战部分的实验，你可以完成挑战方向一节的要求，也可以自行探索；如果可能，请同时编写对应的自动评测脚本。助教将按照你实现的难度给出评分。

如有疑问，参照 `clang -cc1 -dump-tokens 2>&1`。你需要提交一份实验报告，简要记录你的实验过程、遇到的难点以及解决的方法，并在报告中附上自动评测的结果。

### 自动评测细则

本次实验的评测项目为 `lexer-[0-3]`。`lexer-0` 仅用于证明模板（代码与评测脚本）可以正确工作，不计入成绩；其他三个评测项依次检查：

1. `sysu-lexer` 是否提取出正确的 token（60 分）。
2. `sysu-lexer` 是否提取出正确的 token location（30 分）。
3. `sysu-lexer` 是否识别其他无关字符（10 分）。

评测脚本忽略空白符，可以查看[评测脚本](https://github.com/arcsysu/SYsU-lang/blob/latest/compiler/sysu-compiler)以了解检查算法，但不得修改评测逻辑而投机取巧。你也可以像这样调用评测脚本，单独执行其中某一个评测项。

```
( export PATH=$HOME/sysu/bin:$PATH \
  CPATH=$HOME/sysu/include:$CPATH \
  LIBRARY_PATH=$HOME/sysu/lib:$LIBRARY_PATH \
  LD_LIBRARY_PATH=$HOME/sysu/lib:$LD_LIBRARY_PATH &&
  sysu-compiler --unittest=lexer-1 "**/*.sysu.c" )
```

### 挑战方向

本节给出一些挑战方向供参考。

1. 扩展更多 C 语言的 token。
2. 不借助 flex，并完全使用 SYsU 完成本实验，然后用它作为输入测试功能是否正确，以实现自举。
   - 提示：如果你不知道如何下手的话，仔细回忆老师上课所讲的内容，尤其是**正则文法与有限自动机（FA）的等价性**！
3. 借助 libclang 实现相同的功能。
4. 改进这个实验模板（欢迎 PR！）。
5. Do what you want to do。

## 你可能会感兴趣的

- [Lexical Analysis With Flex](http://westes.github.io/flex/manual/)
- [FindFLEX — CMake 3.18.6 Documentation](https://cmake.org/cmake/help/v3.18/module/FindFLEX.html)
- [Preprocessor Output](https://gcc.gnu.org/onlinedocs/gcc-10.2.0/cpp/Preprocessor-Output.html)
- [这篇博客](https://wu-kan.cn/2020/05/14/使用词法分析器-Flex-提取程序中的整数和浮点数/)提到了一种处理注释的方案，如果你不想使用 [flex 自带的注释处理方法](http://westes.github.io/flex/manual/Comments-in-the-Input.html)（当然，实际上注释在预处理阶段已经去除了…）。
- [这篇博客](https://wu-kan.cn/2020/07/03/正则表达式关系判定/)通过构造有限自动机，判断两个正则表达式的关系。
