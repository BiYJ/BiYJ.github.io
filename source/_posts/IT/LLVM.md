---
title: LLVM
categories: IT


## 一、什么是 LLVM？

> The LLVM Project is a collection of modular and reusable compiler and toolchain technologies.

简单来说，LLVM 项目是<font color=#cc0000>一系列分模块、可重用的编译工具链</font>。它提供了一种代码编写良好的中间表示(IR)，可以作为多种语言的后端，还可以提供与变成语言无关的优化和针对多种 cpu 的代码生成功能。

先来看下LLVM架构的主要组成部分：

* 前端：前端用来获取源代码然后将它转变为某种中间表示，我们可以选择不同的编译器来作为LLVM的前端，如 gcc，clang。
* Pass：通常翻译为“流程”，Pass 用来将程序的中间表示之间相互变换。一般情况下，Pass可以用来优化代码，这部分通常是我们关注的部分。
* 后端：后端用来生成实际的机器码。

虽然如今大多数编译器都采用的是这种架构，但是LLVM不同的就是对于不同的语言它都提供了同一种中间表示。

当编译器需要支持多种源代码和目标架构时，基于LLVM的架构，设计一门新的语言只需要去实现一个新的前端就行了，支持新的后端架构也只需要实现一个新的后端就行了。其它部分完成可以复用，就不用再重新设计一次了。


## 二、安装编译LLVM

这里使用 clang 作为前端:

1. 直接从官网下载：[http://releases.llvm.org/download.html](http://releases.llvm.org/download.html)

2. svn 获取

	```
	svn co http://llvm.org/svn/llvm-project/llvm/trunk llvm
	cd llvm/tools
	svn co http://llvm.org/svn/llvm-project/cfe/trunk clang
	cd ../projects
	svn co http://llvm.org/svn/llvm-project/compiler-rt/trunk compiler-rt
	cd ../tools/clang/tools
	svn co http://llvm.org/svn/llvm-project/clang-tools-extra/trunk extra
	```

3. git 获取

	```
	$ git clone http://llvm.org/git/llvm.git
	$ cd llvm/tools
	$ git clone http://llvm.org/git/clang.git
	$ cd ../projects
	$ git clone http://llvm.org/git/compiler-rt.git
	$ cd ../tools/clang/tools
	$ git clone http://llvm.org/git/clang-tools-extra.git
	```

4. 最新的 LLVM 只支持 cmake 来编译了，首先安装 cmake。

	```
	$ brew install cmake
	```
	
5. 编译

	```
	$ mkdir build
	$ cmake /path/to/llvm/source
	$ cmake --build .
	```

	编译时间比较长，而且编译结果会生成 20G 左右的文件。编译完成后，就能在 `build/bin/` 目录下面找到生成的工具了。


## 三、从源码到可执行文件

我们在开发的时候的时候，如果想要生成一个可执行文件或应用，我们点击 run 就完事了，那么在点击 run 之后编译器背后又做了哪些事情呢？

我们先来一个例子：

```
#import <Foundation/Foundation.h>

#define TEN 10

int main(){
    @autoreleasepool {
        int numberOne = TEN;
        int numberTwo = 8;
        NSString* name = [[NSString alloc] initWithUTF8String:"AloneMonkey"];
        int age = numberOne + numberTwo;
        NSLog(@"Hello, %@, Age: %d", name, age);
    }
    return 0;
}
```

上面这个文件，我们可以通过命令行直接编译，然后链接：

```
$ xcrun -sdk iphoneos clang -arch armv7 -F Foundation -fobjc-arc -c main.m -o main.o
$ xcrun -sdk iphoneos clang main.o -arch armv7 -fobjc-arc -framework Foundation -o main
```

拷贝到手机运行：

```
monkeyde-iPhone:/tmp root# ./main

2016-12-19 17:16:34.654 main[2164:213100] Hello, AloneMonkey, Age: 18
```

下面继续深入剖析。

#### 3.1 预处理 Preprocess

这部分包括：

1. macro 宏的展开
2. import/include 头文件的导入
3. \#if 等处理。

可以通过执行以下命令，来告诉 clang 只执行到预处理这一步：

```
$ clang -E main.m
```

执行完这个命令之后，我们会发现导入了很多的头文件内容。

<center>
![](http://dzliving.com/LLVM_0.png)
</center>

可以看到上面的预处理已经把宏替换了，并且导入了头文件。但是这样的话会引入很多不会去改变的系统库比如 Foundation，所以有了 pch 预处理文件，可以在这里去引入一些通用的头文件。

后来 Xcode 新建的项目里面去掉了 pch 文件，引入了 moduels 的概念，把一些通用的库打成 modules 的形式，然后导入，默认会加上 -fmodules 参数。

```
$ clang -E -fmodules main.m
```

这样的话，只需要 @import 一下就能导入对应库的 modules 模块了。

```
@import Foundation; 
int main(){
    @autoreleasepool {
        int numberOne = 10;
        int numberTwo = 8;
        NSString* name = [[NSString alloc] initWithUTF8String:"AloneMonkey"];
        int age = numberOne + numberTwo;
        NSLog(@"Hello, %@, Age: %d", name, age);
    }
    return 0;
}
```

#### 3.2 词法分析 Lexical Analysis

在预处理之后，就要进行词法分析了，将预处理过的代码转化成一个个 Token，比如左括号、右括号、等于、字符串等等。

```
$ clang -fmodules -fsyntax-only -Xclang -dump-tokens main.m
```

<center>
![](http://dzliving.com/LLVM_1.png)
</center>

#### 3.3 语法分析 Semantic Analysis

根据当前语言的语法，验证语法是否正确，并将所有节点组合成抽象语法树(AST)

```
clang -fmodules -fsyntax-only -Xclang -ast-dump main.m
```

<center>
![](http://dzliving.com/LLVM_2.png)
</center>

#### 3.4 IR 代码生成 CodeGen

CodeGen 负责将语法树从顶至下遍历，翻译成 LLVM IR，LLVM IR 是 Frontend 的输出，也是 LLVM Backerend 的输入，桥接前后端。

可以在中间代码层次去做一些优化工作，我们在 Xcode 的编译设置里面也可以设置优化级别 -O1，-O3，-Os。还可以去写一些自己的 Pass，这里需要解释一下什么是 Pass。

Pass 就是 LLVM 系统转化和优化的工作的一个节点，每个节点做一些工作，这些工作加起来就构成了 LLVM 整个系统的优化和转化。

```
$ clang -S -fobjc-arc -emit-llvm main.m -o main.ll
```

<center>
![](http://dzliving.com/LLVM_3.png)
</center>

#### 3.5 生成字节码 LLVM Bitcode

我们在 Xcode7 中默认生成 bitcode 就是这种的中间形式存在，开启了 bitcode，那么苹果后台拿到的就是这种中间代码，苹果可以对 bitcode 做一个进一步的优化，如果有新的后端架构，仍然可以用这份 bitcode 去生成。

```
clang -emit-llvm -c main.m -o main.bc
```

<center>
![](http://dzliving.com/LLVM_4.png)
</center>

#### 3.6 生成相关汇编

```
clang -S -fobjc-arc main.m -o main.s
```

<center>
![](http://dzliving.com/LLVM_5.png)
</center>

#### 3.7 生成目标文件

```
clang -fmodules -c main.m -o main.o
```

<center>
![](http://dzliving.com/LLVM_6.png)
</center>

#### 3.8 生成可执行文件

```
clang main.o -o main
./main
```

<center>
![](http://dzliving.com/LLVM_7.png)
</center>


## 四、可以用 Clang 做什么？

#### 4.1 libclang 进行语法分析

可以使用 libclang 里面提供的方法对源文件进行语法分析，分析它的语法树，遍历语法树上面的每一个节点。可以用于检查拼写错误，或者做字符串加密。

来看一段代码的使用：

```
void * hand = dlopen("/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/libclang.dylib",RTLD_LAZY);
        
//初始化函数指针
initlibfunclist(hand);

CXIndex cxindex = myclang_createIndex(1, 1);

const char *filename = "/path/to/filename";

int index = 0;

const char ** new_command = malloc(10240);

NSMutableString *mus = [NSMutableString stringWithString:@"/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/clang -x objective-c -arch armv7 -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk"]; 

NSArray *arr = [mus componentsSeparatedByString:@" "];

for (NSString *tmp in arr) {
    new_command[index++] = [tmp UTF8String];
}

nameArr = [[NSMutableArray alloc] initWithCapacity:10];

TU = myclang_parseTranslationUnit(cxindex, filename, new_command, index, NULL, 0, myclang_defaultEditingTranslationUnitOptions());

CXCursor rootCursor = myclang_getTranslationUnitCursor(TU);

myclang_visitChildren(rootCursor, printVisitor, NULL);

myclang_disposeTranslationUnit(TU);
myclang_disposeIndex(cxindex);
free(new_command);

dlclose(hand);
```

然后我们就可以在 `printVisitor` 这个函数里面去遍历输入文件的语法树了。

我们也通过 python 去调用用 clang：

```
$ pip install clang
```

那么基于语法树的分析，我们可以针对字符串做加密。


#### 4.2 LibTooling

对语法树有完全的控制权，可以作为一个单独的命令使用，如：clang-format

```
$ clang-format main.m
```

我们也可以自己写一个这样的工具去遍历、访问、甚至修改语法树。 目录:llvm/tools/clang/tools

上面的代码通过遍历语法树，去修改里面的方法名和返回变量名：

```
before:
void do_math(int *x) {
    *x += 5;
}

int main(void) {
    int result = -1, val = 4;
    do_math(&val);
    return result;
}

after:
** Rewrote function def: do_math
** Rewrote function call
** Rewrote ReturnStmt

Found 2 functions.

void add5(int *x) {
    *x += 5;
}

int main(void) {
    int result = -1, val = 4;
    add5(&val);
    return val;
}
```

那么，我们看到 LibTooling 对代码的语法树有完全的控制，那么我们可以基于它去检查命名的规范，甚至做一个代码的转换，比如实现 OC 转 Swift。

#### 4.3 ClangPlugin

对语法树有完全的控制权，作为插件注入到编译流程中，可以影响 build 和决定编译过程。目录:llvm/tools/clang/examples

```
#include "clang/Driver/Options.h"
#include "clang/AST/AST.h"
#include "clang/AST/ASTContext.h"
#include "clang/AST/ASTConsumer.h"
#include "clang/AST/RecursiveASTVisitor.h"
#include "clang/Frontend/ASTConsumers.h"
#include "clang/Frontend/FrontendActions.h"
#include "clang/Frontend/CompilerInstance.h"
#include "clang/Frontend/FrontendPluginRegistry.h"
#include "clang/Rewrite/Core/Rewriter.h"
using namespace std;
using namespace clang;
using namespace llvm;
Rewriter rewriter;
int numFunctions = 0;
class ExampleVisitor : public RecursiveASTVisitor<ExampleVisitor> {
private:
    ASTContext *astContext; // used for getting additional AST info
public:
    explicit ExampleVisitor(CompilerInstance *CI) 
      : astContext(&(CI->getASTContext())) // initialize private members
    {
        rewriter.setSourceMgr(astContext->getSourceManager(), astContext->getLangOpts());
    }
    virtual bool VisitFunctionDecl(FunctionDecl *func) {
        numFunctions++;
        string funcName = func->getNameInfo().getName().getAsString();
        if (funcName == "do_math") {
            rewriter.ReplaceText(func->getLocation(), funcName.length(), "add5");
            errs() << "** Rewrote function def: " << funcName << "\n";
        }    
        return true;
    }
    virtual bool VisitStmt(Stmt *st) {
        if (ReturnStmt *ret = dyn_cast<ReturnStmt>(st)) {
            rewriter.ReplaceText(ret->getRetValue()->getLocStart(), 6, "val");
            errs() << "** Rewrote ReturnStmt\n";
        }        
        if (CallExpr *call = dyn_cast<CallExpr>(st)) {
            rewriter.ReplaceText(call->getLocStart(), 7, "add5");
            errs() << "** Rewrote function call\n";
        }
        return true;
    }
};
class ExampleASTConsumer : public ASTConsumer {
private:
    ExampleVisitor *visitor; // doesn't have to be private
public:
    // override the constructor in order to pass CI
    explicit ExampleASTConsumer(CompilerInstance *CI):
        visitor(new ExampleVisitor(CI)) { } // initialize the visitor
    // override this to call our ExampleVisitor on the entire source file
    virtual void HandleTranslationUnit(ASTContext &Context) {
        /* we can use ASTContext to get the TranslationUnitDecl, which is
             a single Decl that collectively represents the entire source file */
        visitor->TraverseDecl(Context.getTranslationUnitDecl());
    }
};
class PluginExampleAction : public PluginASTAction {
protected:
    // this gets called by Clang when it invokes our Plugin
    // Note that unique pointer is used here.
    std::unique_ptr<ASTConsumer> CreateASTConsumer(CompilerInstance &CI, StringRef file) {
        return llvm::make_unique<ExampleASTConsumer>(&CI);
    }
    // implement this function if you want to parse custom cmd-line args
    bool ParseArgs(const CompilerInstance &CI, const vector<string> &args) {
        return true;
    }
};
static FrontendPluginRegistry::Add<PluginExampleAction> X("-example-plugin", "simple Plugin example");
```
```
clang -Xclang -load -Xclang ../build/lib/PluginExample.dylib -Xclang -plugin -Xclang -example-plugin -c testPlugin.c
** Rewrote function def: do_math
** Rewrote function call
** Rewrote ReturnStmt
```

我们可以基于 ClangPlugin 做些什么事情呢？我们可以用来定义一些编码规范，比如代码风格检查，命名检查等等。下面是我写的判断类名前两个字母是不是大写的例子，如果不是报错。(当然这只是一个例子而已。。。)

## 五、动手写 Pass

#### 5.1 一个简单的 Pass

前面我们说到，Pass 就是 LLVM 系统转化和优化的工作的一个节点，当然我们也可以写一个这样的节点去做一些自己的优化工作或者其它的操作。下面我们来看一下一个简单 Pass 的编写流程：

1. 创建头文件

	```
	$ cd llvm/include/llvm/Transforms/
	$ mkdir Obfuscation
	$ cd Obfuscation
	$ touch SimplePass.h
	```

	写入内容：

	```
	#include "llvm/IR/Function.h"
	#include "llvm/Pass.h"
	#include "llvm/Support/raw_ostream.h"
	#include "llvm/IR/Intrinsics.h"
	#include "llvm/IR/Instructions.h"
	#include "llvm/IR/LegacyPassManager.h"
	#include "llvm/Transforms/IPO/PassManagerBuilder.h"
	
	// Namespace
	using namespace std;
	
	namespace llvm {
		Pass *createSimplePass(bool flag);
	}
	```

2. 创建源文件

	```
	$ cd llvm/lib/Transforms/
	$ mkdir Obfuscation
	$ cd Obfuscation
	
	$ touch CMakeLists.txt
	$ touch LLVMBuild.txt
	$ touch SimplePass.cpp
	```
	
	CMakeLists.txt:

	```
	add_llvm_loadable_module(LLVMObfuscation
	  SimplePass.cpp
	  
	  )
	  
	  add_dependencies(LLVMObfuscation intrinsics_gen)
	```

	LLVMBuild.txt:

	```
	[component_0]
	type = Library
	name = Obfuscation
	parent = Transforms
	library_name = Obfuscation
	```

	SimplePass.cpp:

	```
	#include "llvm/Transforms/Obfuscation/SimplePass.h"
	using namespace llvm;
	namespace {
	    struct SimplePass : public FunctionPass {
	        static char ID; // Pass identification, replacement for typeid
	        bool flag;
	         
	        SimplePass() : FunctionPass(ID) {}
	        SimplePass(bool flag) : FunctionPass(ID) {
	        	this->flag = flag;
	        }
	         
	        bool runOnFunction(Function &F) override {
	        	if(this->flag){
	                Function *tmp = &F;
	                // 遍历函数中的所有基本块
	                for (Function::iterator bb = tmp->begin(); bb != tmp->end(); ++bb) {
	                    // 遍历基本块中的每条指令
	                    for (BasicBlock::iterator inst = bb->begin(); inst != bb->end(); ++inst) {
	                        // 是否是add指令
	                        if (inst->isBinaryOp()) {
	                            if (inst->getOpcode() == Instruction::Add) {
	                                ob_add(cast<BinaryOperator>(inst));
	                            }
	                        }
	                    }
	                }
	            }
	            return false;
	        }
	         
	        // a+b === a-(-b)
	        void ob_add(BinaryOperator *bo) {
	            BinaryOperator *op = NULL;
	             
	            if (bo->getOpcode() == Instruction::Add) {
	                // 生成 (－b)
	                op = BinaryOperator::CreateNeg(bo->getOperand(1), "", bo);
	                // 生成 a-(-b)
	                op = BinaryOperator::Create(Instruction::Sub, bo->getOperand(0), op, "", bo);
	                 
	                op->setHasNoSignedWrap(bo->hasNoSignedWrap());
	                op->setHasNoUnsignedWrap(bo->hasNoUnsignedWrap());
	            }
	             
	            // 替换所有出现该指令的地方
	            bo->replaceAllUsesWith(op);
	        }
	    };
	}
	 
	char SimplePass::ID = 0;
	 
	// 注册pass 命令行选项显示为simplepass
	static RegisterPass<SimplePass> X("simplepass", "this is a Simple Pass");
	Pass *llvm::createSimplePass() { return new SimplePass(); }
	```

	修改 `.../Transforms/LLVMBuild.txt`，加上刚刚写的模块 `Obfuscation`

	```
	subdirectories = Coroutines IPO InstCombine Instrumentation Scalar Utils Vectorize ObjCARC Obfuscation
	```

	修改 `.../Transforms/CMakeLists.txt`，加上刚刚写的模块 `Obfuscation`

	```
	add_subdirectory(Obfuscation)
	```

	编译生成：`LLVMSimplePass.dylib`

	因为 Pass 是作用于中间代码，所以我们首先要生成一份中间代码：

	```
	clang -emit-llvm -c test.c -o test.bc
	```
	
	然后加载 Pass 优化：

	```
	../build/bin/opt -load ../build/lib/LLVMSimplePass.dylib -test < test.bc > after_test.bc
	```
	
	对比中间代码：

	```
	llvm-dis test.bc -o test.ll
	llvm-dis after_test.bc -o after_test.ll
	```
	```
	test.ll
	......
	entry:
	  %retval = alloca i32, align 4
	  %a = alloca i32, align 4
	  %b = alloca i32, align 4
	  %c = alloca i32, align 4
	  store i32 0, i32* %retval, align 4
	  store i32 3, i32* %a, align 4
	  store i32 4, i32* %b, align 4
	  %0 = load i32, i32* %a, align 4
	  %1 = load i32, i32* %b, align 4
	  %add = add nsw i32 %0, %1
	  store i32 %add, i32* %c, align 4
	  %2 = load i32, i32* %c, align 4
	  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %2)
	  ret i32 0
	}
	......
	```
	```
	after_test.ll
	......
	entry:
	  %retval = alloca i32, align 4
	  %a = alloca i32, align 4
	  %b = alloca i32, align 4
	  %c = alloca i32, align 4
	  store i32 0, i32* %retval, align 4
	  store i32 3, i32* %a, align 4
	  store i32 4, i32* %b, align 4
	  %0 = load i32, i32* %a, align 4
	  %1 = load i32, i32* %b, align 4
	  %2 = sub i32 0, %1
	  %3 = sub nsw i32 %0, %2
	  %add = add nsw i32 %0, %1
	  store i32 %3, i32* %c, align 4
	  %4 = load i32, i32* %c, align 4
	  %call = call i32 (i8*, ...) @printf(i8* getelementptr inbounds ([4 x i8], [4 x i8]* @.str, i32 0, i32 0), i32 %4)
	  ret i32 0
	}
	......
	```

	这里写的 Pass 只是把 a+b 简单的替换成了 a-(-b)，只是一个演示，怎么去写自己的 Pass，并且作用于代码。

#### 5.2 将 Pass 加入 PassManager 管理

上面我们是单独去加载 Pass 动态库，这里我们将 Pass 加入 PassManager，这样我们就可以直接通过 clang 的参数去加载我们的 Pass 了。

首先在 `llvm/lib/Transforms/IPO/PassManagerBuilder.cpp` 添加头文件。

```
#include "llvm/Transforms/Obfuscation/SimplePass.h"
```

然后添加如下语句：

```
static cl::opt<bool> SimplePass("simplepass", cl::init(false),
                           cl::desc("Enable simple pass"));
```

然后在 `populateModulePassManager` 这个函数中添加如下代码：

```
MPM.add(createSimplePass(SimplePass));
```
 
最后在 IPO 这个目录的 `LLVMBuild.txt` 中添加库的支持，否则在编译的时候会提示链接错误。具体内容如下：

```
required_libraries = Analysis Core InstCombine IRReader Linker Object ProfileData Scalar Support TransformUtils Vectorize Obfuscation
```

修改 Pass 的 CMakeLists.txt 为静态库形式：

```
add_llvm_library(LLVMObfuscation
  SimplePass.cpp
  )
add_dependencies(LLVMObfuscation intrinsics_gen)
```

最后再编译一次。

那么我们可以这么去调用：

```
../build/bin/clang -mllvm -simplepass test.c -o after_test
```

基于 Pass，我们可以做什么？我们可以编写自己的 Pass 去混淆代码，以增加他人反编译的难度。

我们可以把代码左上角的样子，变成右下角的样子，甚至更加复杂~


## 六、总结

1. LLVM 编译一个源文件的过程：

	<center>
	预处理
	
	↓
	
	词法分析
	
	↓
	
	Token
	
	↓
	
	语法分析
	
	↓
	
	AST

	↓

	代码生成

	↓

	LLVM IR
	
	↓
	
	优化

	↓

	生成汇编代码

	↓

	Link

	↓

	目标文件
	</center>

2. 基于LLVM，我们可以做什么？

	* 做语法树分析，实现语言转换：OC 转 Swift、JS or 其它语言
	* 字符串加密
	* 编写 ClangPlugin，命名规范，代码规范，扩展功能。
	* 编写 Pass，代码混淆优化。


## 文章

[关于LLVM，这些东西你必须要知道](https://neyoufan.github.io/2016/12/29/ios/%E5%85%B3%E4%BA%8ELLVM%E8%BF%99%E4%BA%9B%E4%B8%9C%E8%A5%BF%E4%BD%A0%E5%BF%85%E9%A1%BB%E8%A6%81%E7%9F%A5%E9%81%93%20/)