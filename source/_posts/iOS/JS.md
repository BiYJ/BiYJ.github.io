* 脚本置于 `<body>` 元素的底部，可改善显示速度，因为脚本编译会拖慢显示。
* 外部脚本不能包含 `<script>` 标签。
* 在 HTML 文档完全加载后使用 `document.write()` 将删除所有已有的 HTML 
* JavaScript 语句由值、运算符、表达式、关键词和注释构成。
* JavaScript 语句会按照它们被编写的顺序逐一执行。
* 以 `;` 结束语句不是必需的；
* 首字符必须是字母、下划线`_`或美元符号`$`；
* JavaScript 对大小写敏感；
* JavaScript 使用 `Unicode` 字符集；
* 不带有值的变量，它的值将是 `undefined`；
* 重复声明 JavaScript 变量将不会丢掉它的值；
* 字符串使用加号 `+`、`+=` 将被<font color=#cc0000>级联</font>；

	```
	var x = 3 + 5 + "8";  // 88
	var x = "8" + 3 + 5;  // 835
	```
	
* 基础类型，== 和 === 是有区别的；高级类型没区别；

	> ==	等于；===	等值等型
	
* `**` 幂。`**=` 运算符跨浏览器表现并不稳定，请勿使用

	```
	var x = 2**3;  // 8，与 Math.pow(2, 3) 相同
	var y = 2**4;  // 16 
	```

* `%` 系数运算符
* JavaScript 拥有动态类型。这意味着相同变量可用作不同类型
	
	```
	var x;               // 现在 x 是 undefined
	var x = 7;           // 现在 x 是数值
	var x = "Bill";      // 现在 x 是字符串值
	```

* JavaScript 只有一种数值类型，即没有 float 和 int 的区分
* `typeof` 运算符对数组返回 `object`，因为在 JavaScript 中数组属于对象。
* 任何变量均可通过设置值为 `undefined` 进行清空。其类型也将是 undefined。
* 在 JavaScript 中，`null` 的数据类型是对象，可以通过设置值为 null 清空对象
* Undefined 与 null 的值相等，但类型不相等

	```
	typeof undefined              // undefined
	typeof null                   // object
	null === undefined            // false
	null == undefined             // true
	```
	```
	typeof "Bill"                // 返回 "string"
	typeof 3.14                  // 返回 "number"
	typeof true                  // 返回 "boolean"
	typeof false                 // 返回 "boolean"
	typeof x                     // 返回 "undefined" (假如 x 没有值)
	typeof {name:'Bill', age:62} // 返回 "object"
	typeof [1,2,3,4]             // 返回 "object" (并非 "array"，参见下面的注释)
	typeof null                  // 返回 "object"
	typeof function myFunc(){}   // 返回 "function"
	```
	
* 函数中的代码将在其他代码调用该函数时执行：

	* 当事件发生时（当用户点击按钮时）
	* 当 JavaScript 代码调用时
	* 自动的（自调用

* 对象

	```
	var person = {
	  firstName: "Bill",
	  lastName : "Gates",
	  id       : 678,
	  fullName : function() {
	    return this.firstName + " " + this.lastName;
	  }
	};
	```

* 请不要把字符串、数值和布尔值声明为对象，会增加代码的复杂性并降低执行速度

	```
	var x = new String();        // 把 x 声明为 String 对象
	var y = new Number();        // 把 y 声明为 Number 对象
	var z = new Boolean();       // 把 z 声明为 Boolean 对象
	```

* JavaScript 对象无法进行对比，比较两个 JavaScript 将始终返回 `false`
* 长字符串换行时，某些浏览器也不允许 `\` 字符之后的空格；
* `search()` 方法无法设置第二个开始位置参数；`indexOf()` 方法无法设置更强大的搜索值（正则表达式）
* slice() 两个参数，可以传负数；substring() 两个参数，无法传负数；substr() 两个参数，第二个参数表示截取长度。
* 正则表达式不带引号
* 所有字符串方法都会返回新字符串。它们不会修改原始字符串。正式地说：字符串是不可变的：字符串不能更改，只能替换。
* JavaScript 数值始终是 `64` 位的浮点数
* NaN 属于 JavaScript 保留词，指示某个数不是合法数。NaN 是数，typeof NaN 返回 number
* Infinity 是超出最大可能数范围时返回的值，typeOf Infinity 返回 number。
* 在 JavaScript 中，数组使用数字索引；对象使用命名索引。具有命名索引的数组被称为关联数组
* Switch case 使用严格比较（===）
* JavaScript 中有五种可包含值的数据类型：

	* 字符串（string）
	* 数字（number）
	* 布尔（boolean）
	* 对象（object）
	* 函数（function）

	有三种对象类型：
	
	* 对象（Object）
	* 日期（Date）
	* 数组（Array）

	同时有两种不能包含值的数据类型：
	
	* null
	* undefined
* JavaScript 使用 32 位按位运算数。在执行位运算之前，JavaScript 将数字转换为 32 位<font color=#cc0000>有符号整数</font>。执行按位操作后，结果将转换回 64 位 JavaScript 数。
* \>\> 是保留符号的右移，>>> 是填充 0 的右移。
* 正则表达式是构成搜索模式的字符序列。该搜索模式可用于文本搜索和文本替换操作。`/pattern/modifiers`
* throw 语句抛出异常，异常可以是 JavaScript 字符串、数字、布尔或对象：