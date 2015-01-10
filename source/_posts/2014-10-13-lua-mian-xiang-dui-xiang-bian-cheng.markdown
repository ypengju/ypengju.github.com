---
layout: post
title: "Lua -- 面向对象编程"
date: 2014-10-13 22:08:10 +0800
comments: true
categories: lua
---
在了解表的使用，以及元表和元方法的使用后，就可以用表来实现面向对象编程了。    
###类
面向对象编程的核心就是类了，先来看看lua怎么实现类，并通过类生成不同实例。   <!-- more -->

```lua Class
local Account = {}
function Account:new( o )
	o = o or {}
	setmetatable(o, self)
	self.__index = self
	return o
end
function Account:printBalance(  )
	print("Account balance = " .. self.balance)
end
a = Account:new{ balance = 1 }
b = Account:new{ balance = 2}
a:printBalance() -- Account balance = 1
b:printBalance() -- Account balance = 2
```   
生成了两个实例a，b，都可以调用printBalance()函数打印内容，那么就来分析一下：  

* 首先调用`Accout:new{ balance = 1 }`将一个表传入new()函数中，如果传入参数为空，赋值一个空的表   
* 设置表o的元表为Account，此时的self代表Account   
* 设置Account表的`__index`域为自己，然后返回表o，此时得到一个元表为Account的表    
* 当调用printBalance()表的时候，在返回的表中没有定义此方法，所以会去访问返回表的元表    `__index`域，这个域是Account表,并在表Account定义了printBalance()方法，所以就调用了Account表中定义的printBalance()方法。     
* a:printBalance()时，此时self就是a表，所以self.balance就是a表中的balance，所以就打印1   

这样就定义了一个简单的类，使得new出来的实例，可以调用类中定义的方法。   
###继承
有类就要有继承，再看lua怎么实现继承   
```lua 继承
local Account = { balance = 0 }
function Account:new( o )
	o = o or {}
	setmetatable(o, self)
	self.__index = self
	return o
end

function Account:printBalance(  )
	print("Account balance = " .. self.balance)
end

function Account:addBalance( v )
	if tonumber(v) then
		self.balance = self.balance + v
	end
end

SubAccount = Account:new()

function SubAccount:printSub(  )
	print("this is come from subAccount")
end

a = SubAccount:new()
a:printSub() 		-- this is come from subAccount
a:printBalance()    -- Account balance = 0
a:addBalance(10)
a:printBalance()    -- Account balance = 10   
```   

* 首先定义Account表，相当于父类，其中定义了一个成员变量及两个方法    
* 通过`Account:new()`定义SubAccount，相当于子类，在子类中定义了printSub()方法。    
* `a = SubAccount:new()`，用来生成子类的实例，可是SubAccount中并没有定义new()方法，但可以看到SubAccount就是`Account:new()`返回的一个表，因为传入参数为零，所以此时其实返回的是一个空表，空表中当然没有new()方法，但是这个空表的元表为Account表，并且`__index`域设置为Account表，所以追中`SubAccount:new()`调用的就是`Account:new()`，所以a就是一个表，表的元表就是Account表    
* 当调用printSub()，因为SubAccount定义了printSub，直接调用，相当于子类有这个方法，直接调用子类的方法    
* 调用printBalance()，SubAccount并未定义次方法，所以就会去访问`__index`域，就会调到`Account:printBalance()`方法，而在此方法中，调用了self.balance成员，此时的self是a，a中也没有balance定义，所以就会方位`__index`域，调用`Account.balance`,最后打印出0    
* 调用addBalance(10)，相当于给Account中的balance加了10,所以再次打印的时候为10    
* 上边两个方法的访问相当于子类中没有定义的方法和变量，会去长它的父类。    

###多继承
除了单继承，lua还可以实现多重继承，继续看例子   
```lua 
local A = {}
function A:hello(  )
	print("A hello method ")
end

local B = {}
function B:world(  )
	print("B world method ")
end

function createClass( ... )
	local c = {}
	local parents = { ... }

	local function search( k, parents )
		for i=1, #parents do
			local func = parents[i][k]
			if func then
				return func
			end
		end
	end

	setmetatable(c, {__index = function (t, k)
		return search(k, parents)
	end})

	c.__index = c

	function c:new( o )
		o = o or {}
		setmetatable(o, c)
		return o
	end
	return c
end

C = createClass(A, B)
c1 = C:new()
c1:hello() -- A hello method 
c1:world() -- B world method 
```   

* 先定义基类A,B，并分别定义了了方法    
* 然后使用createClass()方法实现多重继承，其实可以看到，只是遍历两个基类表，找到对应的方法，并调用     
* C相当于继承自A，B，实现了多重继承，来看具体调用过程    
* `c1 = C:new()`生成C的实例，C其实就是createClass()方法中的表c，(具体是c的一个副本)，所以`C:new()`调用的是`function c:new( o )`函数，返回一个以c表为元表的表   
* 调用`c1:hello()`方法，因为`C:new()`返回的表中并未定义hello()方法，所以会去访问反回表的元表`__idnex`域，即`c.__index = c`，多以访问了c表，在c表中，也没有定义hello()方法，所以会继续访问c表的元表`__index`域，可以看到c表元表`__index`与定义的是一个函数，在这个函数中调用了search()函数，search()函数完成的一个功能是根据一个k查找由`createClass(A, B)`传进来参数生成的表parents，遍历parents中所有表，查找为k的值，此时k就是方法名，这个hello()方法会在A中找到，所以返回，所以最终调用了A中的hello()方法。   
* c1:world()同上，调用了B中的world()方法     
* 这样就实现了多重继承，使得子类可以访问父类方法。但这只是简单的模拟实现了多重继承，具体多重继承中的好多问题并未解决。     

###封装
类的一大作用就是封装，使得我们可以选择性的对外暴露的借口。
```lua
function newAccount( initValue )
	local self = { balance = initValue }
	local hello = function (  )
		print("hello method")
	end

	local world = function (  )
		print("world method")
	end

	local getBalance = function (  )
		return self.balance
	end

	return {
		hello = hello,
		getBalanceValue = getBalance
	}
end

n = newAccount(10)
n.hello()
print(n.getBalanceValue())
```   
可以看到在表的最后生成一个新表，只将需要的方法暴露给外边，像上边newAccount中的world()方法就只能在内部访问，外部无法访问。   
[《Programming in Lua》](http://www.lua.org/pil/16.5.html)中讲到一种使用闭包实现更高效的方法。   
```lua 
function newObject( value )
	return function (action, v)
		if action == "get" then return value
		elseif action == "set" then value = v
		else error("invalid action")
		end
	end
end

d = newObject(0)
print(d("get"))    --> 0
d("set", 10)
print(d("get"))    --> 10
```