---
layout: post
title: "Lua -- metatables"
date: 2014-10-04 17:10:37 +0800
comments: true
categories: lua
---
通过lua中的元表及元方法，我们可以定义一些原本在lua中不存在的操作，比如两个表的相加，相减，大小比较，而且lua中的类的继承，类的封装这些面向对象操作也是借助于元表来完成的。   

###设置获取元表
lua中是通过setmetatable()和getmetatable()方法来分别设置和获取一个表的元表的。<!--more-->一个表默认是没有元表的，如下：  
```lua
t = {}
print(getmetatable(t))
```   
打印为nil，说明表t没有元表。但当我们设置元表后，就可以获得元表地址了。   
```lua
t = {}
setmetatable(t, {})
print(getmetatable(t)) -- table: 0x7fc4aa408580
```
###元方法
一个表拥有元表后，就可以添加元方法了。
####算术元方法
```lua
mt = {} --元表
--生成新表
function new(l)
	local set = {}
	setmetatable(set, mt) 	-- 设置元表
	for k,v in pairs(l) do
		set[v] = true
	end
	return set
end
--打印表中元素
function tabletostring(l)
	local res = {}
	for k,v in pairs(l) do
		res[#res + 1] = k
	end
	return "{" .. table.concat(res, ", ") .. "}"
end
--定义 +操作
mt.__add = function ( a , b )
	local res = new{}
	for k in pairs(a) do
		res[k] = true
	end
	for k in pairs(b) do
		res[k] = true
	end
	return res
end
t1 = new{10, 20}
t2 = new{30, 40}
t = t1 + t2
print(tabletostring(t)) -- {20, 30, 40 10}
```  
先申明一个元表mt，new方法用于生成新的表，所有用new方法生成的表都有相同的元表，然后在元表中实现`__add`方法，用于计算表的相加。这里用于将相加的两个表中元素统一到一个新表中。当t1 + t2时，会调用`__add`定义的方法，最后用tabletostring方法打印新表中的内容，这里可以看到，新表中统一了两个表的元素。   
当lua调用`+`操作时，首先查找左参数是否定义了`__add`域，如果定义了，调用这个`__add`操作，如果没定义，查找第二个参数，是否定义了`__add`域，如果定义了，调用第二个参数的`__add`域，如果两个参数都没有定义，就会报错。  
与__add域一样，还可以定义其他域来完成表的`-`,`*`,`/`等操作 

* __sub 减
* __mul 乘
* __div 除
* __mod 取余
* __unm 负值

####逻辑元方法
除了定义算数运算外，还可以定义逻辑运算，用来比较两个表的大小，但是lua中只制定了`__eq`,`__lt`,`__le`来分别代表，等于、小于、小于等于这三种操作，像其他的操作，比如不等，大于等操作，lua会基于上边三种进行转换，比如不等就是 not (a==b)。
```lua 小于等于
mt.__le = function ( a, b )
	for k,v in pairs(a) do
		if  not b[k] then return false end
	end
	return true
end
t1 = new{10, 20, 30}
t2 = new{10, 20}
print(t1 <= t2) -- false
print(t2 <= t1) -- true
```   
遍历a表，如果a对应元素b表都有，说明a表小于等于b表，否则不小于等于。则根据上边，小于可定义如下
```lua 小于
mt.__le = function ( a, b )
	for k,v in pairs(a) do
		if  not b[k] then return false end
	end
	return true
end
mt.__lt = function ( a, b )
	return a <= b and not (b <= a)
end
t1 = new{10, 20, 30}
t2 = new{10, 20}
print(t1 < t2)  --false
print(t2 < t1)	--true
print(t1 > t2)  --true
print(t1 >= t2) --true
```   
如果a<b，则a<=b必然成立，但b<=a肯定不成立。同时可以看到可以用`>`、`>=`操作，这是因为lua帮我们自动转换了。
```lua 等于
mt.__eq = function ( a, b )
	return a <= b and b <= a
end
t1 = new{10, 20, 30}
t2 = new{10, 20}
print(t1 == t2) --false
print(t1 ~= t2) --true 
```
同时我们可以根据小于等于快速定义等于操作，此时我们也可以使用`~=`不等于操作。   
####库定义元方法
```lua
print({}) -- table: 0x7fad7bc03c80
```
默认情况下，使用print函数单元一个表的时候，打印的是他的地址。我们可以定义元表中的`__tostring`域来自定义打印的内容。   
```lua
function tabletostring(l)
	local res = {}
	for k,v in pairs(l) do
		res[#res + 1] = k
	end
	return "{" .. table.concat(res, ", ") .. "}"
end
mt.__tostring = tabletostring
t1 = new{10, 20, 30}
print(t1) --{10, 20, 30}   
```   
将tabletostring方法赋值给`__tostring`域，当我们再次使用print函数打印new生成的表的时候，就讲真个表的内容打印出来了。    
```
t1 = new{10, 20, 30}
print(getmetatable(t1)) --table: 0x7f9851408c20
setmetatable(t1, {})    
print(getmetatable(t1)) --table: 0x7faef2409ea0
```   
像上边这样，当再次给原来的表设置元表后，原来的设置的元表就失效了，为了防止这种情况发生，使得设置一次元表后不能更改，我们可以实现元表的`__metatable`域，来防止元表被更改。
```
mt.__metatable = "this is metatable"
t1 = new{10, 20, 30}
print(getmetatable(t1)) --this is metatable
setmetatable(t1, {})  -- 报错 cannot change a protected metatable
```   
当调用getmetatable时，将放回我们定义的`__metatable`域的值，当再次设置元表时，就会报错，提示元表是被保护的。   
####表访问元方法
当访问表中不存在的元素时，lua会返回nil，其实我们可以利用元表，自定义返回的值。
```
mt.__index = function (_, k)
	return 300
end
t1 = new{10, 20, 30}
print(t1[5]) -- 300
```  
像上边这样，当访问下标为5的元素时，元表中是没有的，本来应该返回的是nil，但当我们定义了`__index`域时，返回值就是300了。   
`__index`的定义有两种，一种是像上边这样使用函数，一种是使用表   
```
t2 = {100, 200, 300, 400, 500, 600}
mt.__index = t2
t1 = new{10, 20, 30}
print(t1[5]) -- 500
```   
像上边这样直接使用表定义`__index`域，当访问t1中下标为5的元素时，t1中并不存在，但我们定义元表中的`__index`域，此时就会访问t2中下标为5的元素，并返回，所以t1[5]返回的值就是500。  
还可以定义`__newindex`域操作给表中不存在的元素赋值时的操作。
```lua
t1 = new{10, 20, 30}
mt.__newindex = function ()
	print("new index")
end
t1[5] = 1 -- new index
```   
当对不存在的元素赋值时，就会调用`__newindex`域定义的方法。   

像上边这样用new生成的所有新表都共享同一个元表，这就有点像c++中类生成无数实例一样，而访问不存在的元素，就会查找`__index`域，这就有点像在c++中，访问子类没有的方法，就去父类查找。而lua就是用这种特性来实现面向对象编程的。

