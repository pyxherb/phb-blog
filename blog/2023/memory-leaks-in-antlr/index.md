# ANTLR for C++在VC++下提示内存泄漏的解决方案

## 问题背景

用ANTLR4替代Flex/Bison来开发语法分析部分后，程序每次退出时总会提示内存泄漏（参见[这个issue](https://github.com/antlr/antlr4/issues/4309)）：

```txt
Detected memory leaks!
Dumping objects ->
{11750} normal block at 0x0000000000436E90, 128 bytes long.
 Data: <  C       C     > 90 04 43 00 00 00 00 00 90 04 43 00 00 00 00 00 
{11749} normal block at 0x00000000004202B0, 16 bytes long.
 Data: <`iC             > 60 69 43 00 00 00 00 00 00 00 00 00 00 00 00 00 
{11748} normal block at 0x0000000000430490, 24 bytes long.
 Data: <  C       C     > 90 04 43 00 00 00 00 00 90 04 43 00 00 00 00 00 
{11747} normal block at 0x000000000041FDB0, 16 bytes long.
 Data: <HiC             > 48 69 43 00 00 00 00 00 00 00 00 00 00 00 00 00 
{11746} normal block at 0x0000000000436F50, 128 bytes long.
 Data: <  C       C     > D0 09 43 00 00 00 00 00 D0 09 43 00 00 00 00 00 
{11745} normal block at 0x000000000041FB30, 16 bytes long.
 Data: < hC             > E8 68 43 00 00 00 00 00 00 00 00 00 00 00 00 00 
{11744} normal block at 0x00000000004309D0, 24 bytes long.
 Data: <  C       C     > D0 09 43 00 00 00 00 00 D0 09 43 00 00 00 00 00 
{11743} normal block at 0x000000000041F310, 16 bytes long.
 Data: < hC             > D0 68 43 00 00 00 00 00 00 00 00 00 00 00 00 00 
{11742} normal block at 0x0000000000439B90, 128 bytes long.
 Data: <0 C     0 C     > 30 04 43 00 00 00 00 00 30 04 43 00 00 00 00 00 
{11741} normal block at 0x000000000041FBD0, 16 bytes long.
 Data: <phC             > 70 68 43 00 00 00 00 00 00 00 00 00 00 00 00 00 
(...)
```

## 定位/解决问题

在ANTLR中添加VC++内存检测支持后重新编译，定位到内存泄漏来源于以下源文件：

```txt
runtime/Cpp/runtime/src/atn/LexerATNSimulator.cpp(192)
runtime/Cpp/runtime/src/atn/LexerATNSimulator.cpp(295)
runtime/Cpp/runtime/src/atn/LexerATNSimulator.cpp(536)
runtime/Cpp/runtime/src/atn/ParserATNSimulator.cpp(299)
runtime/Cpp/runtime/src/atn/ParserATNSimulator.cpp(465)
runtime/Cpp/runtime/src/atn/ParserATNSimulator.cpp(531)
runtime/Cpp/runtime/src/atn/ParserATNSimulator.cpp(618)
runtime/Cpp/runtime/src/atn/ParserATNSimulator.cpp(636)
runtime/Cpp/runtime/src/dfa/DFA.cpp(29)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(179)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(182)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(185)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(188)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(191)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(194)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(197)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(200)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(203)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(206)
runtime/Cpp/runtime/src/atn/ATNDeserializer.cpp(212)
runtime/Cpp/runtime/src/atn/ATNDeserializationOptions.cpp(17)
runtime/Cpp/runtime/src/atn/LexerMoreAction.cpp(16)
runtime/Cpp/runtime/src/atn/LexerSkipAction.cpp(16)
runtime/Cpp/runtime/src/atn/LexerPopModeAction.cpp(16)
```

其中大部分泄漏都是与ATN相关的，根据引用顺藤摸瓜发现ATN中提示泄漏的这些东西最终被存储到了DFA缓存中，而DFA缓存最终又被放在lexer和parser的静态数据中（供lexer和parser使用，只会在初次构造lexer和parser时各自初始化一次）：

```cpp
// 本文中采用的Parser均名为MyParser，下文不再赘述
// 用来存储Parser的静态数据的结构体，Lexer也有
struct MyParserStaticData final {
  MyParserStaticData(std::vector<std::string> ruleNames,
                        std::vector<std::string> literalNames,
                        std::vector<std::string> symbolicNames)
      : ruleNames(std::move(ruleNames)), literalNames(std::move(literalNames)),
        symbolicNames(std::move(symbolicNames)),
        vocabulary(this->literalNames, this->symbolicNames) {}

  MyParserStaticData(const MyParserStaticData&) = delete;
  MyParserStaticData(MyParserStaticData&&) = delete;
  MyParserStaticData& operator=(const MyParserStaticData&) = delete;
  MyParserStaticData& operator=(MyParserStaticData&&) = delete;

  std::vector<antlr4::dfa::DFA> decisionToDFA; // DFA缓存就存储在这里
  antlr4::atn::PredictionContextCache sharedContextCache;
  const std::vector<std::string> ruleNames;
  const std::vector<std::string> literalNames;
  const std::vector<std::string> symbolicNames;
  const antlr4::dfa::Vocabulary vocabulary;
  antlr4::atn::SerializedATNView serializedATN;
  std::unique_ptr<antlr4::atn::ATN> atn;
};

// Parser的静态数据，只会在Parser初次构建时初始化一次
MyParserStaticData* myparserParserStaticData;
```

继续查找发现Parser和Lexer的静态数据是由名为xxxxParserInitialize的函数初始化的（xxxx为你的parser名称），而静态数据直接使用了原始指针存储，而且在别处没有发现任何释放代码：

```cpp
MyParserStaticData* myparserParserStaticData;
```

为了验证内存泄漏是否是因为静态数据没有被正确释放产生的，这里使用了unique_ptr替代原始指针存储parser和lexer的静态数据进行测试（以下仅演示Parser部分，Lexer情况和Parser类似）：

```cpp
std::unique_ptr<MyParserStaticData> myparserParserStaticData;
```

为了让修改后的代码能正常工作，我们还需要修改xxxxParserInitialize函数：

```cpp
myparserParserStaticData = staticData.release();
```

这句需要改成

```cpp
myparserParserStaticData = std::move(staticData);
```

对Lexer和Parser都进行操作之后重新编译运行，大部分内存泄漏消失：

```txt
Detected memory leaks!
Dumping objects ->
C:\Users\Pyxherb\Desktop\antlr4\runtime\Cpp\runtime\src\atn\ATNDeserializationOptions.cpp(17) : {434} normal block at 0x000002891574C2C0, 3 bytes long.新
 Data: <   > 00 01 00 
Object dump complete.
```

（剩下的一条内存泄漏是由于ANTLR内部某个方法为了返回左值而new了一个对象导致的，目前暂时还未解决）

为了使修改能够一直有效（不至于每次从g4文件生成parser和lexer后都得编辑一次源码），我们需要修改ANTLR生成C++ Parser和Lexer用的模板文件，我这里采用了修改后自己重新构建一份jar的方法：

1. clone一份ANTLR的repo
2. 找到tool/resources/org/antlr/v4/tool/templates/codegen/Cpp/Cpp.stg（生成C++的Parser和Lexer用的模板文件）
3. 按照上述修改C++代码的步骤修改模板文件，具体操作可以看[这个commit](https://github.com/antlr/antlr4/pull/4374/commits/7c670e07537884f7acf9e9e512fd91d202fb3da7)
4. 重新编译ANTLR，用编译出来的jar（建议选取名称中带有-complete的）替换原先的jar

------------------------------

2023年首发于CSDN
