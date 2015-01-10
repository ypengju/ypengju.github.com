---
layout: post
title: "cocos2d-x -- rapidjson"
date: 2014-09-28 16:10:30 +0800
comments: true
categories: cocos2d-x
---
rapidjson是一个c++中的json解析库，其实还有其他的工具如jsoncpp，但是号称rapidjson效率更高一些，并且cocos2d-x3.x版本中都添加rapidjson支持，所以学习了解了下rapidjson的使用。  
#####rapidjson 主页：[https://code.google.com/p/rapidjson/](https://code.google.com/p/rapidjson/)   

下载后在源码中example中可以找到使用方法，在cocos2d-x 3.2中，位于external/json   <!-- more -->

* document.h     
* filestream.h   
* prettywriter.h  
* rapidjson.h   
* reader.h   
* stringbuffer.h   
* writer.h   

在使用的时候，我们根据需要,包含进来我们需要的文件，以下是在cocos2d-x中的使用：  

``` c++ 引入头文件
#include "../cocos2d/external/json/rapidjson.h"   
#include "../cocos2d/external/json/document.h"   
#include "../cocos2d/external/json/stringbuffer.h"   
#include "../cocos2d/external/json/writer.h"   
#include "../cocos2d/external/json/filestream.h"   
#include "../cocos2d/external/json/prettywriter.h"   
using namespace rapidjson;  
```   
##解析
rapidjson中的所有类型都位于rapidjson命名空间中。以下根据官方使用说明，例句一些具体的使用例子，帮助理解。   
``` c++ 简单解析  
const char json[] = "{ \"hello\" : \"world\" }";   
rapidjson::Document doc;   
doc.Parse<kParseDefaultFlags>(json); 
printf("%s \n", doc["hello"].GetString());   
```    
运行程序，在输出台中就可以看到打印的 `world` 了。上边的例子将一个json文本解析成为一个DOM树，其中Parse<kParseDefaultFlags>,中的kParseDefaultFlags是一个默认解析标志。解析的数据默认是以UTF-8的编码格式存储，同时rapidjson还支持UTF-16,UTF-32。   
####Document 类型   
Document代表json对象，我们可以下边方式生成一个rapidjson队像。  
``` c++ json对象
rapidjson::Document doc;   
```    
####Value 类型
Value是用于存储json值或者键值对，它可以存储false， true， number， string， array或者object等数据格式。上边讲到的Document类型其实派生自Value。   
下边来看个例子：
json数据   
{   
　　　"hello":"world",   
　　　"t":true,   
　　　"f":false,   
　　　"n":null,   
　　　"i":123,   
　　　"pi":3.1416,   
　　　"a":[1,2,3,4]   
}
``` c++ 解析例子
 
    const char json1[] = " { \"hello\" : \"world\", \"t\" : true , \"f\" : false, \"n\": null, \"i\":123, \"pi\": 3.1416, \"a\":[1, 2, 3, 4] } ";
    
    //解析数据到document中
    rapidjson::Document document;
    document.Parse<kParseDefaultFlags>(json1);
    assert(document.IsObject());
    assert(document.HasMember("hello"));
    assert(document["hello"].IsString());
    printf("hello = %s\n", document["hello"].GetString());
    
    assert(document["t"].IsBool());
    printf("t = %s\n", document["t"].GetBool() ? "true" : "false");
    
    assert(document["f"].IsBool());
    printf("f = %s\n", document["f"].GetBool() ? "true" : "false");
    
    printf("n = %s\n", document["n"].IsNull() ? "null" : "?");
    
    assert(document["i"].IsNumber()); // Number is a JSON type, but C++ needs more specific type.
    assert(document["i"].IsInt());	  // In this case, IsUint()/IsInt64()/IsUInt64() also return true.
    printf("i = %d\n", document["i"].GetInt());	// Alternative (int)document["i"]
    
    assert(document["pi"].IsNumber());
    assert(document["pi"].IsDouble());
    printf("pi = %g\n", document["pi"].GetDouble());
    
    //解析数组
    {
         const rapidjson::Value& a = document["a"]; //handy and faster.
         assert(a.IsArray());
        for (SizeType i = 0; i < a.Size(); i++)	{// rapidjson uses SizeType instead of size_t.
             printf("a[%d] = %d\n", i, a[i].GetInt());
         }
    }
```   
此例可在官方例子程序中找到，尤其注意程序中对于数组成员的访问，我们不能直接使用下标访问数组成员，因为[]操作符在c++中是模糊不清的，它也可代表一个const char *的空指针，如果这样写，编辑器会直接提示错误，如上所示，我们在rapidjson中访问数组成员变量时，对类型进行包装，即SizeType(0)，或者在下标后添加u字符， SizeType在rapidjson中代表size_t类型。   
运行程序，可在输出看到如下打印：   
hello = world   
t = true   
f = false   
n = null   
i = 123   
pi = 3.1416   
a[0] = 1   
a[1] = 2   
a[2] = 3   
a[3] = 4   
此外，调用HasMember()方法可以确定是否包含此值。  
####数据遍历
上边已经说过，因为0在c++中可以代表数值也可以代表空指针，所以我们在遍历数组成员的时候使用下边形式代替：   

* a[SizeType(0)]
* a[0u]

其实我们还可以像遍历std::vector一样，用迭代器，遍历rapidjson数组成员。   
``` c++ 遍历数组
for (rapidjson::Value::ConstValueIterator itr = a.onBegin(); itr != a.onEnd(); ++itr) {
     printf("-------%d\n", itr->GetInt());
}    
```
#####官方例子给的是a.Begin()和a.End();但在cocos2d-x3.2中使用时调用的是a.onBegin()和a.onEnd()  
####遍历对象
除了对数组可以使用迭代器进行遍历外，我们还可以用迭代器对对象进行遍历，如下：   
``` c++   遍历对象
for (rapidjson::Value::ConstMemberIterator itr = document.MemberonBegin(); itr != 				document.MemberonEnd(); ++itr)
{
        printf("-----type of member %s \n", itr->name.GetString());
}
```   
#####此处注意是document.MemberonBegin()而不是官方文档中的document.MemberBegin()   
在rapidjson中可以判断一个数是是有符号整数，还是无符号整数，是单精度数还是双精度数等方法，如下：  

* IsNumber() whether the value is a number
* IsInt() whether the number is a int
* IsUint() whether the number is a uint
* IsInt64() whether the number is a int64_t
* IsUint64() whether the number is a uint64_t
* IsDouble() whether the number is a double

对于字符串的操作，rapidjson除了可以使用getString()获取字符内容外，还可以使用getStringLength()获取字符串长度。   
##生成
先来看个简单的例子：   
``` c++ 
rapidjson::Document docWrite;   
docWrite.SetObject();   
//    或者
//    rapidjson::Value doc;
//    doc.SetObject();
FileStream f(stdout);
PrettyWriter<FileStream> writer(f);
docWrite.Accept(writer);   
```   
上边我们生成了一个空对象，并把它写到控制台中，这样我们就可以在控制台中看到一个{}了，我们可以用两种方式来生成一个对象，即rapidjson::Document 或者 rapidjson::Value，因为Document是派生自Value的，所以Value能干的事，它也能干。至于写入流程，现在还不是懂，等懂了再说。  
除了这种SetXXX()的方式外，我们换可以使用构造的方式来完成，rapidjson::Value doc(kObjectType);   
####对象操作：

如果给对象添加成员时，我们可以调用AddMember()方法在其中添加成员，与其对应的还有一个RemoveMember()方法用于删除对象中成员。
```c++ 对象添加成员
Document doc;
doc.SetObject();  
doc.AddMember("name", "Milo", doc.GetAllocator());
doc.AddMember("married", true, doc.GetAllocator());    
//    doc.RemoveMember("married");    
FileStream f(stdout);
PrettyWriter<FileStream> writer(f);
doc.Accept(writer);
```
生成格式：   
{   
　　　"name": "Milo",   
　　　"married": true   
}
####字符串操作   
rapidjson对于字符串的操作提供了两种方案。   

1. copy-string   
2. const-string  
   
第一种是将原字符串拷贝了一份，然后，对拷贝后的字符串进行操作，这样做安全很多，但是对于长字符串就比较耗时间了。   
另一种只是存储了一份指向原字符串的指针，这样就大大减少了对内存使用。    
```
Document docWriteStr;
docWriteStr.SetObject();
rapidjson::Value author;
char buffer[10];
int len = sprintf(buffer, "%s %s", "Milo", "Yip"); // dynamically created string.
author.SetString(buffer, len, docWriteStr.GetAllocator());
memset(buffer, 0, sizeof(buffer));
docWriteStr.AddMember("author", author, docWriteStr.GetAllocator());
FileStream fs(stdout);
PrettyWriter<FileStream> writerStr(fs);
docWriteStr.Accept(writerStr);  
```

这种在SetString()的时候，先使用零时buffer，对字符传进行了拷贝，计算字符长度，然后使用doc.GetAllocator()进行了内存分配。生成Value成员，然后使用AddMember方法将其添加到docWriteStr对象中。

另一方面，我们不需要生成零时字符串，可直接对字符串进行操作。
``` c++ 
rapidjson::Value author;
author.SetString("rapidjson", 9);
```
这种方式比上边的效率更高。还可以使用下边这两种形式，但由于不知道字符长度，效率会比较低一点。  
```
author.SetString("rapidjson");
author = "rapidjson";
```
####数组操作：
``` c++ 生成数组
Document docWriteArray;
docWriteArray.SetObject();
rapidjson::Value array(kArrayType);
for (int i = 0; i < 5; ++i) {
	array.PushBack(i, docWriteArray.GetAllocator());
}
docWriteArray.AddMember("array", array, doc.GetAllocator());
FileStream fArray(stdout);
PrettyWriter<FileStream> writerArray(fArray);
docWriteArray.Accept(writerArray);
```
生成格式：   
{   
　　　"array": [0,1,2,3,4]   
}    
在rapidjson::Value array(kArrayType);中，使用kArrayType声明Value类型为数组，我们也可以使用 array.SetArray();的方式。然后使用PushBack()方法将值加载到数组中，最后将Value添加到对象中。   
如果我们想在数组中生成对象，只要将生成新的Value,然后添加就行，像下边这样：
``` c++ 数组中添加对象
Document doc;
doc.SetObject();
rapidjson::Value array(kArrayType);
for (int i = 0; i < 3; ++i) {
    rapidjson::Value tmp(kObjectType);
    tmp.AddMember("class1", "ypengju", doc.GetAllocator());
    tmp.AddMember("class2", "xcode", doc.GetAllocator());
    array.PushBack(tmp, doc.GetAllocator());
}    
doc.AddMember("array", array, doc.GetAllocator());
FileStream f(stdout);
PrettyWriter<FileStream> writer(f);
doc.Accept(writer);
```
生成格式：   
{   
　　　"array": [   
　　　　　　{   
　　　　　　　　　"class1": "ypengju",   
　　　　　　　　　"class2": "xcode"   
　　　　　　},   
　　　　　　{   
　　　　　　　　　"class1": "ypengju",   
　　　　　　　　　"class2": "xcode"   
　　　　　　},   
　　　　　　{   
　　　　　　　　　"class1": "ypengju",   
　　　　　　　　　"class2": "xcode"   
　　　　　　}   
　　　]   
}   

#####例子程序地址：[https://github.com/ypengju/cocos2d-xTest](https://github.com/ypengju/cocos2d-xTest)