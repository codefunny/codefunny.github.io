---
layout: post
title: "Javascript学习笔记"
date: 2014-11-27 18:08:26 +0800
comments: true
categories: javascript
---

Javascript学习笔记

<!-- more -->

#javascript模块化编写

	event = function(){
		return {
			bind: function(){},
			unbind: function(){},
			trigger: function(){}
		};
	}();


可以参考jQuery的匿名函数执行写法

	(function(window){
		//..
		// exports
		window.jQuery = window.$ = jQuery;
	})(window);

Seajs的写法

	this.seajs = { _seajs: this.seajs };


其实上面的this也可以去掉，如Ext

	Ext = {
		version : '3.1.0'
	};
下面代码经过在chrome浏览器测试可以运行

	//jQuery写法
	;(function(window){
	
		animate = {
			Cat:Cat
		}
	
		function Cat(name,color){
			this.name = name;
			this.color = color;
			this.type = '猫科动物';
			this.eat = function(){ alert('mouse');};
		}
	
		window.animate = animate;
	
	})(window);
	
	//执行函数之后，myevent就是{}对象了。
	myevent = function(){
		function _show(){
			alert('this is show!');
		}
	
		return {
			show : _show
		};
	}();
	
	//seajs写法
	this.peterjs = {_peterjs: this.peterjs};
	peterjs = {
		show : function(){
			alert('this is peterjs');
		}
	};
	
	peterjs.version = '1.1.0';
	
	//ext写法
	Ext = {
		version : '3.1.0'
	};

【参考】

1. http://www.cnblogs.com/snandy/archive/2012/03/08/2378441.html

#Javascript面向对象方法

####constructor属性和instanceof方法
constructor属性指向它们的构造函数；<br/>
instanceof运算符，验证原型对象与实例对象之间的关系；

####prototype模式
javascript规定，每一个构造函数都有一个prototype属性，指向另一个对象。这个对象的所有属性和方法，都会被构造函数的实例继承。意味着，可以将那些不变的属性和方法，直接定义在prototype对象上。

	function Cat(name,color){
		this.name = name;
		this.color = color;
	}
	//注意是类名不是实例名
	Cat.prototype.type = '猫科动物';
	Cat.prototype.eat = function(){ alert('eatmouse'); };
	
	var cat1 = new Cat('damao','orange');
	alert(cat1.type);
	cat1.eat();
	

####isPrototypeOf()和hasOwnProperty()方法与in运算符
1. isPrototype()用来判断某个prototype对象和某个实例之间的关系
2. hasOwnProperty()用来判断某一个属性到底是本地属性还是继承自prototype对象的属性.
3. 用于判断某个实例是否含有某个属性，不管是不是本地属性

下面代码帮助理解：

	alert(Cat.prototype.isPrototypeOf(cat1)); //true
	alert(cat1.hasOwnProperty('name')); //true
	alert(cat1.hasOwnProperty('type')); //false
	alert('name' in cat1); //true
	alert('type' in cat1); //true
	
【参考】
1. http://www.ruanyifeng.com/blog/2010/05/object-oriented_javascript_inheritance.html

#Javascript构造函数的继承
对象之间继承的5种方法

	function Animal(){
		this.species = "animate";
	}
	
	function Cat(name,color){
		this.name = name;
		this.color = color;
	}

一、 构造函数绑定
使用call或apple方法，将父对象的构造函数绑定在子对象上

	function Cat(name,color){
		Animal.apply(this, arguments);
		this.name = name;
		this.color = color;
	}
	
	var cat1 = new Cat('tom','blue');
	alert(cat1.species);

二、prototype模式

	Cat.prototype = new Animal();
	Cat.prototype.constructor = Cat;
	var cat1 = new Cat('tome','blue');
	alert(cat1.species);

对上面需要说明的是第二行，因为第一行将Cat的prototype指向了一个新的对象，那么相应地prototype.constructor也就指向了Animal，不是我们要的结果，所以要加上第二行纠正。

三、直接继承prototype（这个方法是有问题的）
该方法是第二种方法的改进。由于Animal对象中不变的属性都可以直接写入Animal.prototype中，所以可以直接继承Animal.prototype。

	function Animal(){}
	Animal.prototype.species = "Animal";
	Cat.prototype = Animal.prototype;
	Cat.prototype.constructor = Cat;

上面的代码执行效率比较高，因为不用执行和建立Animal的实例。但是有一个缺点就是，任何对Cat。prototype的修改都会反映到Animal.prototype上。就是说第四行实际上也把Animal的构造函数给改了。

四、利用空对象作为中介（这是对上面方法的改进）

	function extend(Child,Parent){
		var F = function(){};
		F.prototype = Parent.prototype;
		Child.prototype = new F();
		Child.prototype.constructor = Child;
		Child.uber = Parent.prototype;
	}
	
	extend(Cat,Animal);
	var cat1 = new Cat('tom','blue');
	alert(cat1.species);

上面F是空对象，几乎不占内存。修改Cat的prototype对象，就不会影响到Animal的prototype对象。uber是备用，指向府对象的原型。

五、拷贝继承

	function extend1(Child,Parent){
		var p = Parent.prototype;
		var c = Child.prototype;
		for(var i in p){
			c[i] = p[i];
		}
		
		c.uber = p;
	}
	extend1(Cat,Animal);
	var cat1 = new Cat('tom','blue');
	alert(cat1.species);

#Javascript非构造函数继承

深拷贝(jQuery库使用的就是这种继承方法)

	function deepCopy(p){
		var c = c || {};
		for(var i in p){
			if(typeof p[i] === 'object'){
			   c[i] = (p[i].constructor === Array) ? [] : {};
				deepCopy(p[i],c[i]);
			} else {
				c[i] = p[i];
			}
		}
		return c;
	}

