##一、GLSL基础

1. GLSL（OpenGL Shading Language）作为一种着色语言是纯粹的和 <font color=#CC00000>GPU</font> 打交道的计算机语言。因为<font color=#CC00000> GPU 是多线程并行处理器</font>，所以 GLSL 直接面向 SIMD 模型的多线程计算。

2. GLSL 编写的着色器函数是对每个数据<font color=#CC00000>同时执行</font>的。
3. 每个顶点都会由顶点着色器中的算法处理，每个像素也都会由片段着色器中的算法处理。

	因此，初学者在编写自己的着色器时，需要考虑到SIMD的并发特性，并用并行计算的思路来思考问题。 

	最常见用法是在顶点着色器里生成所需要的值，然后传给片断着色器用。
	
	<br/>
  

##二、GLSL能做什么？

- 日以逼真的材质 -- 金属，岩石，木头，油漆等
- 日益逼真的光照效果 -- 区域光和软阴影
- 非现实材质 -- 美术效果，钢笔画，水墨画和对插画技术的模拟
- 针对纹理内存的新用途
- 更少的纹理访问 
- 图形处理 -- 选择，边缘钝化遮蔽和复杂混合
- 动画效果 -- 关键帧插值，粒子系统
- 用户可编程的反走样方法

<br/>

##三、GLSL注意
- GLSL支持函数重载
- GLSL不存在数据类型的自动提升，数据类型必须严格保持一致。
- GLSL不支持指针，字符串，字符，它基本上是一种处理数字数据的语言
- GLSL不支持联合、枚举类型、结构体位字段及按位运算符

<br/>

##四、数据类型
GLSL 有三种基本数据类型：<font color=#CC00000>float</font>，<font color=#CC00000>int</font> 和 <font color=#CC00000>bool</font>，以及由这些数据类型组成的<font color=#CC00000>数组</font>和<font color=#CC00000>结构体</font>。

需要注意的是，GLSL 并<font color=#CC00000>不支持指针</font>。与 C/C++ 不同的是，<font color=#CC00000>GLSL 将向量和矩阵作为基本数据类型</font>。

注意：GLSL不存在数据类型的自动提升，类型必须严格保持一致。

####1、标量

- float 
- int 
- bool 

```
42   // 十进制  
042  // 八进制  
0x2A // 十六进制
```
GLSL 不存在数据类型的自动提升，类型必须严格保持一致

<br>

####2、矢量

矢量可以和标量甚至矩阵做加减乘除(必须符合规则)

```
 vec2,  vec3,  vec4 // 包含2/3/4个浮点数的矢量
ivec2, ivec3, ivec4 // 包含2/3/4个整数的矢量
bvec2, bvec3, bvec4 // 包含2/3/4个布尔值的矢量
```

声明：

```
vec3 v;             //声明三维浮点型向量v  
v[1]=3.0;           //给向量v的第二个元素赋值  

// 下面两种等价
vec3 v = vec3(0.6);
vec3 v = vec3(0.6, 0.6, 0.6);
```

注意：除了用索引的方式外，还可以用选择运算符的方式来使用向量。选择运算符是对于向量的各个元素（最多为 <font color=#CC00000>4</font> 个）约定俗成的名称，用一个小写拉丁字母来表示。根据向量表示对象的意义不同，可以使用以下选择运算符：

	表示顶点可以用 (x、y、z、w)

	表示颜色可以用 (r、g、b、a)

	表示纹理坐标用 (s、t、r、q)

用户可以选择其中任意一种选择运算符，它们的作用是等效的。

例如：

```
vec4 v1=vec4(1.0, 2.0, 3.0, 4.0)；   //用构造函数的方式声明并初始化四维浮点型
vec4 v2；  
v2.xy=v1.yz;                    //将v1的第二个和第三个元素复制到v2的第一个和第二个元素 
v2.z=2.0;                       //给v2的第三个元素赋值  
v2.xy=v1.yx;                    //将v1的头两个元素互换，再复制到v2的头两个元素中
```

<br>

####3、矩阵

	mat2, mat3, mat4 -- 2*2 / 3*3 / 4*4 的矩阵

矩阵是按列顺序组织的，<font color=#CC00000>先列后行</font>。

例如：

```
mat4 m;             // 声明四维浮点型方阵 m  
m[2][3]=2.0;        // 给方阵的第三列、第四行元素赋值 

// 下面两种等价，初始化矩阵对角
mat2 m = mat2(1.0)
mat2 m = mat2(1.0, 0.0, 0.0, 1.0);
```

<br>

####4、取样器(Sampler)


纹理查找需要指定哪个纹理或者纹理单元将指定查找。

```
sampler1D           访问一个一维纹理
sampler2D           访问一个二维纹理           
sampler3D           访问一个三维纹理
samplerCube         访问一个立方贴图纹理
sampler1DShadow     访问一个带对比的一维深度纹理
sampler2DShadow     访问一个带对比的二维深度纹理
```

```
uniform sampler2D grass;

vcc2 coord = vec2(100, 100);
vec4 color = texture2D(grass, coord);
// 如果一个着色器要在程序里结合多个纹理，可以使用取样器数组

const int tex_nums = 4;
uniform sampler2D textures[tex_nums];

for(int i = 0; i < tex_nums; ++i) {
    sampler2D tex = textures[i];
    // todo ...
}
```
<br>

####5、结构体

这是<font color=#CC00000>唯一的用户定义类型</font>。

```
struct light  
{  
    vec3 position;  
    vec3 color;  
};  

light ceiling_light;
```

</br>

####6、数组

数组索引是从 0 开始的，而且没有指针概念。

```
// 创建一个具有 10 个元素的数组  
vec4 points[10];  

// 创建一个不指定大小的数组
vec4 points[]; 
points[2] = vec4(1.0);  // points 现在数组长度为 3
points[7] = vec4(2.0);  // points 现在数组长度为 8
```

</br>

####7、void

只能用于声明函数返回值。

</br>

##五、类型转换
必须明确地进行类型转换，不会自动类型提升。

```
float f = 2.3; 
bool b = bool(f); // b is true
```

</br>

##六、限定符
GLSL 中有 4 个限定符（variable qualifiers）可供使用，它们限定了被标记的变量不能被更改的"范围"。

- const
- attribute
- uniform
- varying


####1、const

const 和 C++ 里差不多，定义不可变常量。表示限定的变量在编译时不可被修改

</br>

####2、attribute

attribute 是应用程序传给顶点着色器用的。<font color=#CC00000>不允许声明时初始化</font>。

attribute 限定符标记的是一种全局变量，该变量在顶点着色器中是只读（read-only）的，该变量被用作从OpenGL应用程序向顶点着色器中传递参数，因此该限定符<font color=#CC0000>仅能用于顶点着色器</font>。

</br>

####3、uniform

unifrom 一般是应用程序用于设定顶点着色器和片断着色器相关初始化值。

不允许声明时初始化

uniform限定符标记的是一种全局变量,该变量对于一个图元（primitive）来说是不可更改的 它可以从OpenGL应用程序中接收传递来的参数。

</br>

####4、varying

varying用于传递顶点着色器的值给片断着色器

<font color=#CC00000>不允许声明时初始化</font>

它提供了从顶点着色器向片段着色器传递数据的方法，varying限定符可以在顶点着色器中定义变量，然后再传递给光栅化器，光栅化器对数据插值后，再将每个片段的值交给片段着色器。 

</br>

##七、限制
- 不能在 if-else 中声明变量
- 用于判断的条件必须是bool类型(if,while,for...)
- (?:) 操作符后两个参数必须类型相同
- 不支持 switch 语句

```
vec4 toonify(in float intensify) 
{
    vec4 color;
    color = vec4(0.8,0.8,0.8,0.8)
    return color;
}
```

</br>

## 八、discard关键字

discard 关键字可以避免片段更新帧缓冲区，当流控制遇到这个关键字时，正在处理的片段就会被标记为丢

</br>

## 九、函数
- 函数名可以通过参数类型重载，但是和返回值类型无关
- 所有参数必须完全匹配，参数不会自动
- 函数不能被递归调用
- 函数返回值不能是数组

函数参数标示符

	in: 进复制到函数中，但不返回的参数(默认)

	out: 不将参数复制到函数中，但返回参数

	inout: 复制到函数中并返回 

</br>

## 十、混合操作

通过在选择器 (.) 后列出各分量名，就可以选择这些分量。

```
vec4 v4;
v4.rgba;    // 得到vec4
v4.rgb;     // 得到vec3
v4.b;       // 得到float
v4.xy;      // 得到vec2
v4.xgba;    // 错误！分量名不是同一类

v4.wxyz;    // 打乱原有分量顺序
v4.xxyy;    // 重复分量
```

</br>

#### 原文地址：[https://www.tuicool.com/articles/yEBFvmA](https://www.tuicool.com/articles/yEBFvmA)