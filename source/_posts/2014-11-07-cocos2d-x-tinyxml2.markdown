---
layout: post
title: "cocos2d-x -- tinyxml2"
date: 2014-11-07 22:27:16 +0800
comments: true
categories: cocos2d-x
---
tinyxml2是一个简单，高效的xml语言的库，它是一个开源项目，项目代码托管在github上，这个是它的github地址： [tinyxml2](https://github.com/leethomason/tinyxml2)，在cocos2d-x中，已经集成了tinyxml2库，所以在使用的时候我们只要引入相应的类就可开始处理xml文件。在cocos2d-x中我们经常会用到的CCUserDefault操作者本地的一个xml文件，其实内部实现就有封装tinyxml2来实现的。<!-- more -->

tinyxml2库位于external文件夹下，在tinyxml2中其实就一个.h和一个.cpp文件，非常简单。打开tinyxml2.h文件就可以看到下边几个类
```c++
class XMLDocument;       //文件节点既根节点
class XMLElement;        //元素节点  如<dic></dic>
class XMLAttribute;      //属性值   如<dic version="1.0"></dic>
class XMLComment;        //注释	   如<!--注释-->
class XMLNode;           //XMLDocument，XMLAttribute，XMLComment，XMLText，XMLDeclaration，XMLUnknown的父节点
class XMLText;			 //值       如<dic version="1.0">the text</dic>
class XMLDeclaration;	 //xml开头的声明，用于声明文件格式，版本信息，及编码   如<?xml version="1.0" encoding="UTF-8"?>
class XMLUnknown;		 //<!unknown>标签
```   
这几个类就是我们直接操作的xml文件的元素节点及内容。接下来看看具体如何用tinyxml2生成一个xml文件。
###生成XML文件
```c++ 生成xml文件
void HelloWorld::addXML()
{
    XMLDocument *doc = new XMLDocument();
    if (nullptr == doc) {
        return;
    }
    
    XMLDeclaration *dec = doc->NewDeclaration();
    if (nullptr == dec) { return; }
    doc->LinkEndChild(dec);
    
    //根节点
    XMLElement *root = doc->NewElement("root");
    root->SetAttribute("version", "1.0");
    doc->LinkEndChild(root);
    
    XMLElement *theKey = doc->NewElement("key");
    theKey->LinkEndChild(doc->NewText("key1"));
    root->LinkEndChild(theKey);
    
    XMLElement *theValue = doc->NewElement("value");
    theValue->LinkEndChild(doc->NewText("100"));
    root->LinkEndChild(theValue);
    
    XMLComment *comment = doc->NewComment("数组");
    root->LinkEndChild(comment);
    
    XMLElement *array = doc->NewElement("array");
    root->LinkEndChild(array);
    
    for (int i = 0; i < 4; i++) {
        XMLElement *data = doc->NewElement("data");
        data->LinkEndChild(doc->NewText("arrayData"));
        array->LinkEndChild(data);
    }
    //将xml文件存储在home目录下的text.xml中
    doc->SaveFile("~/text.xml");
    //在控制台打印
    doc->Print();
}
```   
运行程序就可以在控制台打印信息：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<root version="1.0">
    <key>key1</key>
    <value>100</value>
    <!--数组-->
    <array>
        <data>arrayData</data>
        <data>arrayData</data>
        <data>arrayData</data>
        <data>arrayData</data>
    </array>
</root>
```  
首先使用`XMLDocument`生成文件，然后使用`NewDeclaration()`添加文件说明，在说明中我们可以传入字符串，指定格式，版本，字符编码等，如果为空，采取如上默认声明。之后就可以添加元素节点，所有节点都是由XMLDocument生成的，同时可进行嵌套。最后将生成的xml保存到text.xml文件中同时在控制台打印，就如同上边看到的一样。   
###解析xml文件
如上生成的xml文件保存在home目录的text.xml文件中，接下来我们在读取这个文件，然后在程序的解析使用
```c++
void HelloWorld::parseXML()
{
    //解析text.xml文件到doc中
    XMLDocument *doc = new XMLDocument();
    doc->LoadFile("/Users/ypengju/text.xml");
    
    XMLElement *root = doc->RootElement();

    const XMLAttribute *firstAttr = root->FirstAttribute();
    log("--name %s, value = %s", firstAttr->Name(), firstAttr->Value());
    
    XMLElement *first = root->FirstChildElement();
    log("--- first element: name %s, text %s", first->Name(), first->GetText());
    
    XMLElement *second = first->NextSiblingElement();
    log("--- second element: name %s, text %s", second->Name(), second->GetText());

    XMLElement *array = second->NextSiblingElement();
    log("----array element: name %s", array->Name());
    
    for (XMLElement *ele = array->FirstChildElement(); ele != array->LastChildElement()->NextSiblingElement(); ele = ele->NextSiblingElement()) {
        log("----- name %s, value %s", ele->Name(), ele->GetText());
    }
}
```   
控制台打印：  

```
cocos2d: --name version, value = 1.0
cocos2d: --- first element: name key, text key1
cocos2d: --- second element: name value, text 100
cocos2d: ----array element: name array
cocos2d: ----- name data, value arrayData1
cocos2d: ----- name data, value arrayData2
cocos2d: ----- name data, value arrayData3
cocos2d: ----- name data, value arrayData4
```  
首先根据路径加载xml文件，将其存在`XMLDocument`中，然后用其一次获得每个节点及属性值，这样我们在程序中就可以使用了。   
以上就是简单的tinyxml2的生成和解析xml文件。   


