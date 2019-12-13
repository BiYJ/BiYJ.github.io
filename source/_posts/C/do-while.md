---
title: do{...}while(0)
---

## 一、帮助定义复杂的宏以避免错误

假设你需要定义这样一个宏：

```
#define DOSOMETHING()  foo1(); foo2();
```

这个宏的本意是：当调用 DOSOMETHING() 时，函数 foo1() 和 foo2() 都会被调用。但是如果你在调用的时候这么写：

```
if(a > 0)
	DOSOMETHING();
```

因为宏在预处理的时候会直接被展开，你实际上写的代码是这个样子的：

```
if(a>0)
	foo1();
	foo2();
```

这就出现了问题。因为无论 a 是否大于 0，foo2() 都会被执行，导致程序出错。

那么仅仅使用 {} 将 foo1() 和 foo2() 包起来行么？比如：

```
#define DOSOMETHING() { foo1(); foo2(); }
```

我们在写代码的时候都习惯在语句右面加上分号，如果在宏中使用 {}，代码里就相当于这样写了：`{...};`，展开后就是这个样子：

```
if(a > 0)
{
	foo1();
	foo2();
};
```

很明显，这是一个语法错误（花括号后多了一个分号，虽然不会报错）。

在所有可能情况下，期望我们写的多语句宏总能有正确的表现几乎是不可能的。你不能在没有 do/while(0)的情况下让宏表现的像函数一样。如果我们使用 do{...}while(0) 来定义宏，即：

```
#define DOSOMETHING() \
do{ \
	foo1();\
	foo2();\
}while(0)\
```

这样，宏被展开后，上面的调用语句才会保留初始的语义。do 能确保大括号里的逻辑能被执行，而 while(0) 能确保该逻辑只被执行一次，就像没有循环语句一样。

总结：在 Linux 和其它代码库里的，很多宏实现都使用 do/while(0) 来包裹他们的逻辑，这样不管在调用代码中怎么使用分号和大括号，而该宏总能确保其行为是一致的。

如果你开发了一个库，甚至就只是一个宏实现提供给别人使用，你并不能确信所有人都会按照你的意图正确使用该实现，有的使用者处处加大括号，也有的人认为你会处理的。因此你需要保证你的实现在各种使用场景下都是表现一致的。另外，就像 Scott Meyers 在 Effective C++ 中谈及的，让接口容易被正确使用，而不易被误用。 


## 二、避免使用 goto 控制程序流

在一些函数中，我们可能需要在 return 语句之前做一些清理工作，比如释放在函数开始处由 malloc 申请的内存空间，使用 goto 总是一种简单的方法：

```
int foo()
{
	somestruct * ptr = malloc(...);
	...;

	if(error)
		goto END;
	...;  // dosomething
	if(error)
		goto END;
	...;  // dosomething

	END:
		free(ptr);
		return 0;
}
```

但由于 goto 不符合软件工程的结构化，而且有可能使得代码难懂，所以很多人都不倡导使用，这个时候我们可以使用 do{...}while(0) 来做同样的事情：

```
int foo()
{
	somestruct * ptr = malloc(...);
	do
	{
		...;   // dosomething
		if(error)
			break;
		...;   // dosomething
		if(error)
			break;
		...;   // dosomething
	} while(0);

	free(ptr);
	return 0;
}
```

将函数主体部分使用 do{...}while(0) 包含起来，使用 `break` 来代替 goto，后续的清理工作在 while 之后，现在既能达到同样的效果，而且代码的可读性、可维护性都要比上面的 goto 代码好的多了。

比如有个人认为这样会导致很多变量要在 do-while 语句外面提前声明(这是 C++ 程序反对的一点，用时才声明)，也有人认为还有强大的 RAII 和智能指针神马的，不怕内存问题。但是，在某些场景下，要是这些都不能用呢，比如 C 语言？比如没有 smart ptr？比如维护的是旧代码？


## 三、避免由宏引起的警告

内核中由于不同架构的限制，很多时候会用到空宏。在编译的时候，这些空宏会给出 warning，为了避免这样的 warning，我们可以使用 do{...}while(0) 来定义空宏：

```
#define EMPTYMICRO do{}while(0)
```


## 四、定义单一的函数块来完成复杂的操作

如果你有一个复杂的函数，变量很多，而且你不想要增加新的函数，可以使用 do{...}while(0)，将你的代码写在里面，里面可以定义变量而不用考虑变量名会同函数之前或者之后的重复。

这种情况应该是指一个变量多处使用(但每处的意义还不同)，我们可以在每个 do-while 中缩小作用域，比如：

```
int key;
string value;
int func()
{
	int key = GetKey();
	string value = GetValue();

	...;  // dosomething for key, value;
    
	do {
		int key;
		string value;
		...;  // dosomething for this key, value;
	} while(0);    
}
```