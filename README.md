# JavaScript 隐式类型转换浅析 

### 1. 概述 

javascript是一种弱类型语言，每个变量是没有类型限制的，可以进行随意的赋值。
但数据是有类型的，在各种运算中，javascript中引擎会自动将各种类型进行转换，这就是所谓的
隐式转换。如果稍有不注意，转换产生的结果可能并不符合预期，某些情况下很难发现，
又可能有一些不常见的数据出现导致简单的测试检测不出问题。因此了解常见的隐式的转换
有助于写出更稳定的代码。  
    通常我们用于判断数据类型使用的方法是typeof。它返回的类型包括：  
    `"number"` , `"string"` ,  `"boolean"` , `"object"` , `"function"`, `"undefined"` 和 `"symbol"`  
    symbol类型是es6中出现，目前浏览器兼容性还达不到要求，也无法通过babel等工具进行转换为es5的代码，
因此在目前来说使用量较长，也就不在讨论。  
对于各种类型，有都一些特殊的数据，
其中  
> "number"类型包括NaN、Infinity、-Infinity  
> "string"类型中空字符串  
> "object"类型中包括对象(包括一些预置的全局对象)、数组、Null(类型实际上并非object类型，遗留问题)  
> "undefined"本身也是一种特殊的类型。

在各种运算符和一些语句下，这些类型会产生隐式的转换。
大部分转换中，都是调用了浏览器的getValue方法进行了转换。对于不同的类型转换结果也不同。
比如转换为数字：

    Number('')// 0    
    Number('1dd') // NaN    
    Number(true)// 1    


实际上转换过程是比较复杂的，具体的转换规则可以参见ECMA定义的规则。
下面分享一些比较实用的内容。
### 2. false类型的数据
在这些被Boolean()转换后产生false的数据有：
false, '', 0, null, undefined, NaN

### 3. 数学运算中出现的转换。
调用Number函数可将变量显式转换为数据类型，对于特殊类型的数据：转换结果如下：  

	Number('')// 0    
	Number(true)// 1  
	Number(false)// 0 
	Number(null) // 0    
	Number(undefined)// NaN  
	Number(NaN)// NaN  

	Number('1dd') // NaN    
	Number({})// NaN    
	Number([])// 0    
	Number([1])// 1    
	Number([1, 2])// NaN   

	var a = new Number(1)
	typeof a// "object"
	Number(a)// 1 
	  
	Number.prototype.fn = function(){return this}
	var a = 1
	typeof a// "number"
	Number(a)// 1
	typeof a.fn()// "object"
	Number(a.fn())// 1


总结起来就是: false类型的数据，只有undefined和NaN被转换为NaN，其他均转为0，true被转换为1，仅有一个元素的数组则
取出数组中的仅有元素进行转换，字符串中只有数字则可以转换为对应的数字，其他非数字类型均被转换为NaN。  
但对象中有一些特殊的内容，如果使用Number构建的数字，或者用prototype返回this,实际上是一个对象，在转换时会使用构建时的值，而不是普通对象返回NaN。  

实际上，在转换为数字过程中调用了浏览器的getValue方法，这个方法在针对对象类型的数据时，会调用对象的
valueOf或toString方法，

	var a = {
		toString: function(){
			console.log('toString');
			return 1
		}
	}
	Number(a)// toString  1

而加入了valueOf后则会

	var a = {
		toString: function(){
		console.log('toString');
		return 1
		},
		valueOf: function(){
			console.log('toValue');
			return "2"}
		}
	Number(a)// toValue  2
	String(a)// toString  "1"
  
	var a = {
		valueOf: function(){
			console.log('toValue');
			return "2"}
		}
	String(a)// "[object Object]"

从中可以看出，转换为数字时会优先调用valueOf，如果不存在，就会调用toString方法，然后再将
返回值转换为数字。而转换为字符串时则会调用toString,再将返回值转换为字符串，没有toString方法也不会调用valueOf。	

数学运算符的比较多一些，包括一元的 ++ -- + - 
及 +-*/,实质上都是调用了getValue来获取转换为数字的。

适当的运用类型转换可减少代码量。比如我们常用的一个表达式：  
+new Date();  
将日期对象转换为毫秒数为单位的数字，可直接进行运算，相当于 new Date().getTime()
注意，这个是一元运算符不能和二元的混淆，如

	1 + new Date()// "1Wed Nov 09 2016 18:31:17 GMT+0800 (中国标准时间)"  

不会转换为数字类型


### 4. 字符串相加  
运算符 + 是一个比较特殊符号，既可表示转换为正值，也可以表示两个数值相加，还可以表示字符串相加。
我们来看下字符串和其他类型相加产生的结果
  
	console.log("hello" + 1) // "hello1"
	console.log("hello" + true) // "hellotrue"
	console.log('hello' + {})// "hello[object Object]"
	console.log('hello' + Math)// "hello[object Math]"，全局对象中JSON，Math，window，Reflect
	console.log('hello' + Number)// "hellofunction Number() { [native code] }"，Promise返回的是具体代码
	 
	console.log('hello' + null)// "hellonull"
	console.log('hello' + undefined)// "helloundefined"
	 
	console.log('hello'+ NaN)// "helloNaN"
	console.log('hello'+ Infinity)// "helloInfinity"

很普通是吧，和我们预想中的没有什么差别。
根据前面我们为对象定义的方法，

	var a = {
		toString: function(){
		console.log('toString');
		return 1
		},
		valueOf: function(){
			console.log('toValue');
			return "2"
		}
	}
	a + 1// toValue  "21"
	a + 1// toValue  "21"
	// 而我们返回数字时
	var a = {
		toString: function(){
		console.log('toString');
		return 1
		},
		valueOf: function(){
			console.log('toValue');
			return 2
		}
	}
	a+1// toValue  3
  
可见，对象类型与普通类型究竟是字符串加还是数字相加，取决于valueOf返回的数据类型，而不是
对象类型。数组呢？
   
	var a = [1, 2, 3]
	a +1// "1,2,31"
	a.valueOf = function(){console.log('toValue');return 1}
	a+1//  toValue	2

	a.toString = function(){console.log('toString');return 1}
	a + 1// toValue  2

数组实际上只是对象的一种，对象默认返回的数据类型是字符串型，与其他类型相加必然是字符串相加，
除非我们为它指定一个valueOf方法返回指定数据。

实际上在+相加时是调用了ToPrimitive方法，它的定义：
 ToPrimitive 运算符接受一个值，和一个可选的 期望类型 作参数。ToPrimitive 运算符把其值参数转换为非对象类型。如果对象有能力被转换为不止一种原语类型，可以使用可选的 期望类型 来暗示那个类型。根据下表完成转换：

ToPrimitive转换
输入类型	结果
Undefined	结果等于输入的参数（不转换）。
Null	结果等于输入的参数（不转换）。
Boolean	结果等于输入的参数（不转换）。
Number	结果等于输入的参数（不转换）。
String	结果等于输入的参数（不转换）。
Object	返回该对象的默认值。（调用该对象的内部方法[[DefaultValue]]一樣）。

### 5. 各种比较

前面讲了各种false类型，下面讲讲比较的用法。

#### 1. 大于、小于、大于等于、小于等于
对于两个操作数均是字符串的时候&无法转换时的返回值会有不同。
当两个操作数均是字符串的时候，它会执行大家熟悉的字符串比较，即从左到右依次比较每一个字符的ASCII码，若出现符合操作符的情况，则返回true，否则返回false。
无法将操作数转换为数字的情况下总是返回false。
  
	var a = []
	var b = {}
	var c = 1

	a>b// false
	a>c// false

	var a = {
		toString: function(){
		console.log('toString');
		return 1
		},
		valueOf: function(){
			console.log('toValue');
			return 2
		}
	}
	a>0// toValue   true

#### 2. ==  

在== 的符号下，转换中
0. '', false 相等，
而null == undefined
NaN也是false类型，但它不和任何false相等
引用类型的转换比较的是地址，因此只有同一对象的引用会才相等，否则不会相等。
  

		console.log('' ==0)// true
		console.log('' == false)// true
		console.log(0 == false)// true
		console.log(0 == null)// false
		console.log(0 == undefined)// false
		console.log(null == undefined)//  true


### 6.常见的隐式转换函数

#### 1. isNaN

	var a = {
		valueOf: function(){
		console.log('toValue');
		return 2
		}, 
		toString: function(){
		console.log('toString');
		return '0'}
	}
	console.log(isNaN(a))// toValue false


在使用isNaN进行检测的时候会先进行数字的转换，然后再判断是否是NaN
用这个函数用来判断变量是否是NaN是不靠谱的，适用的场景可以在使用了Number
或者parseFloat之后的值进行判断。
想要判断变量是否为NaN，需要使用NaN的一个特性：和任何值均不相等，如
  
	console.log(NaN == NaN)// false

将变量与自身进行比较就可以得出是否是NaN，为方便可将其封装为一个函数

	function elementIsNaN(a){
		return a !== a;
	}
	elementIsNaN(a)// false
	elementIsNaN(NaN)// true

除了这个方法ECMAScript 6中, 有一个Number.isNaN() 方法提供可靠的NaN值检测，该方法在新版的chrome浏览器
中已经支持。 
  
	console.log(Number.isNaN(NaN)); // true
	console.log(Number.isNaN(Math.sqrt(-2))); // true
	console.log(Number.isNaN('hello')); // false
	console.log(Number.isNaN(['x'])); // false
	console.log(Number.isNaN({})); // false

#### 2. if while && || 等真值运算

它们的转换是转变量转换为布尔型，相当于调用Boolean，
（有些书中也推荐在for循环中使用倒序，能够实现真值运算，据说能加快速度）
前面提到的false类型转换后均为false。
而对于对象来说

	var a = {
		valueOf: function(){
		console.log('toValue');
		return 2
		}, 
		toString: function(){
		console.log('toString');
		return '0'}
	}
	if(a)console.log('a is true')/ "a is true"

前面我们看到，调用valueOf时均会显示toValue，而if并没有调用a的valueOf，
这是因为引用类型均会返回true。

我们可以利用这一点来判断函数的传参，如果我们要判断该参数有没有传入，
可以使用：

	typeof a === 'undefined'
或者 
  
	a === undefined

null和undefined具有一些特殊性，如果我们针对目标函数调用方法如toString(),.length等，会产生错误
影响代码可靠性，
有些时候我们会特意传入null来表示该参数不可用，此时无需区分null和undefined，可以利用
null和undefined在 == 下产生隐式转换的特性进行一起判断
 
	if(a == null)console.log(a)

或 

	if(a == undefined)console.log(a)

如果只是判断变量是否为false值，无须用 === 或== 进行判断，简单的写法更能减少代码量

	a = a || "0"
这种方法常用于设置变量的默认值。

非特意的情况下，建议不要使用 == 进行变量值的判断，而是显式使用相应的转换函数
进行转换后用 === 号进行判断

#### 3. 调用toString方法的alert

alert大概是我们在调试过程中用的特别多的一个函数了。它和各种运算不同，不调用toValue,而是调用toString


	var a={
		valueOf: function(){
			console.log('toValue');
			return 2
		}, 
		toString: function(){
			console.log('toString');
			return '0'
		}
	}
	alert(a)// toString "0"

	var a={
		valueOf: function(){
			console.log('toValue');
			return 2
		}
	}
	alert(a)// "[object Object]"

	String.prototype.fn = function(){return this}; 
	var a = 'hello'; 
	alert(typeof a.fn()); //-->object 
	alert(a.fn()); //-->hello 

和前面提到的字符串相加结果是相同的。

#### 4. 对象解析时产生的转换

在Js语法中定义变量，在解析过程中都会进行转换，如常的对象中
{
	a:1,
	b:1,
}
可正常解析为对象，会自动去除多余的逗号。很多规范都建议每个键值后面都加逗号，防止
新添属性或方法时出现缺少逗号的错误
上面写的对象实际上并非JSON对象的规范格式，规范中所有的键名都要以双引号引起来，实际上
解析器在存储过程中已经进行了转换。

	var a = {1: ''}
	JSON.stringify(a)//"{"1":""}"

	for(key in a){
	  console.log(typeof key)
	}// string

	var a = {1: '', true: 2, aaa: 3}
	console.log(a)

将a log出来，可以看到，数字和bool值颜色和字符串型相同。

数组本质也上也是一种对象，其键值的存储规则和对象完全相同，只是数字被转换为了字符串。
  
	var person = {'name':'jack',"age":20,school:'PKU'}; 
	for(var a in person){ 
	    alert(a + ": " + typeof a);// string 
	} 

而在取用数组的值时，也会自动转换为字符串后再取出键值。

	a = [1, 2, 3]
	a['4'] = 222
	a[4]//222


当然了，symbol类型不太相同

	var a = {
	  [mySymbol]: 'symbol',
	  mySymbol: 'string'
	}
	a[mySymbol]// "symbol"
	a['mySymbol']// "string"
	console.log(a)
	Object
	mySymbol:"string"
	Symbol(mysymbol):"symbol"
	__proto__:Object

	var a = {
    [mySymbol]: 'symbol',
    "Symbol(mysymbol)":'fake symbol',
    mySymbol: 'string'
	}
	a[mySymbol]//"symbol"

	console.log(a)
	symbol(mysymbol):"fake symbol"
	mySymbol:"string"
	Symbol(mysymbol):"symbol"

	JSON.stringify(a)
	// "{"Symbol(mysymbol)":"fake symbol","mySymbol":"string"}"

	var a = {
	  [mySymbol]: 'symbol',
	  mySymbol: 'string'
	}
	JSON.stringify(a)// "{"mySymbol":"string"}"

可以看出，尽管我们使用字符串来替代它显示的样式，symbol类型还是完全与普通的变量类型不同的
它无法被JSON.stringify转换为字符串，也就说明实际上对句中Symbol类型的存储并非字符串


#### 5. 转换为number的两类函数

对于数字类型转换的方法，parseInt, parseFloat, Number
其中Number对于false类型的数据，只有undefined和NaN被转换为NaN，其他均转为0，
true被转换为1，仅有一个元素的数组则
取出数组中的仅有元素进行转换，字符串中只有数字则可以转换为对应的数字，其他非数字类型均被转换为NaN，
对象类型则是调用它的valueOf或toString方法了。

而parseInt,parseFloat两个函数，在转换时调用toString方法

	var a = {
		toString: function(){
		console.log('toString');
		return 1
		},
		valueOf: function(){
			console.log('toValue');
			return "2"
		}
	}
	parseFloat(a)// toString  1

再针对字符串进行转换。字符串转换时会从第一个字符算起，如果遇到非数字字符，
停止，将返回前一部分转换为数字的结果
所以对于false类型的数据，只有0能被转换为数字，其他全返回NaN。
使用的时候一定要注意


### 7. 我们能做些什么
1. 熟悉各种类型转换，避免类型错误有可能会被类型转换所隐藏

2. 无法确定参数各类，想进行数学计算的，又不想报出错误，不妨使用按位取反符号，~~

3. "+" 既可以表示字符串连接，又可以表示算术加，这取决于它的操作数，如果有一个为字符串的，那么，
就是字符串连接了，而在带有对象类型的进行加减时，要注意valueOf返回的数据类型，决定了是
算术相加还是字符串连接

4. 具有valueOf方法的对象，应该定义一个相应的toString方法，用来返回相等的数字的字符串形式

5. 对于函数参数判断或者比较，尤其与false类型的比较，一定要分清期望值，尽量用 === !==代替
== !=。

6. 检测一些未定义的变量时，应该使用typeOf或者与undefined作比较，而不应该直接用真值运算

7. 变量类型啥的，还是参考成熟代码更为靠谱

8. 尽量使用显示转换Number, String