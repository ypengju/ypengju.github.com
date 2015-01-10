---
layout: post
title: "cocos2d-x -- LuaEngine调用Lua代码"
date: 2014-11-12 21:39:06 +0800
comments: true
categories: cocos2d-x
---
cocos2d-x 3.2中，在命令行生成新项目时，我们需要使用`-l`参数指明使用的语言，如果使用c++语言，则默认不包含脚本库，如果使用lua语言，则项目默认生成lua环境，我们直接可以开始编写lua逻辑，无需关心引擎到lua的调用工作。为了了解引擎如何从c++调用到lua，z这篇文章以c++项目为例。    <!-- more --> 
对新生成的c++项目，首先需要添加脚本库， 将引擎cocos目录下的scripting目录拷贝到新项目的cocos目录下与引擎原始目录结构保持一致即可。除了scripting还需要将external目录下的lua文件夹也一并考过来放在对应目录，需要的文件拷贝好之后，需要在项目中进行配置，右击项目->"Add File to ...",然后如下图添加脚本库   
{% img  /images/lua/1.png %}    
{% img  /images/lua/2.png %}    
添加成功后，如下将库文件添加到工程
{% img  /images/lua/3.png %} 
分别在"Target Dependence"和"Link Binary With Library"下，这里需要添加另外一个"MediaPlayer.framework",否则会编译出错。   
添加好库之后，还需指明搜索路径
{% img  /images/lua/4.png %}
这样在项目中就可以直接调用`#include "CCLuaEngine.h"`包含头文件，通过LuaEngine调用lua文件了。
```
auto engine = LuaEngine::getInstance();
ScriptEngineManager::getInstance()->setScriptEngine(engine);
if (engine->executeScriptFile("hello.lua")) {
    return false;
}
```   
cocos2d-x支持JS脚本和lua脚本，使用ScriptEngineManager来管理脚本引擎，如上边这个例子，我们获得lua引擎，然后将其设置到脚本管理类中，先来看看这个管理到底是干什么用的。   
```c++ ScriptEngineManager
class CC_DLL ScriptEngineManager
{
public:
    ~ScriptEngineManager(void);
    ScriptEngineProtocol* getScriptEngine(void) {
        return _scriptEngine;
    }
    void setScriptEngine(ScriptEngineProtocol *scriptEngine);
    void removeScriptEngine(void);
    static ScriptEngineManager* getInstance();
    static void destroyInstance();
    static bool sendNodeEventToJS(Node* node, int action);
    static bool sendNodeEventToJSExtended(Node* node, int action);
    static void sendNodeEventToLua(Node* node, int action);
private:
    ScriptEngineManager(void)
    : _scriptEngine(nullptr)
    {
    }
    ScriptEngineProtocol *_scriptEngine;
};
```   
ScriptEngineManager就是一个单例，帮助引擎保存ScriptEngineProtocol对象，ScriptEngineProtocol是LuaEngine的父类，类中声明一些纯虚方法。具体操作都由LuaEngine实现。其实LuaEngine也只是外层的一个包装，其中管理者一个LuaStack对象，所有栈操作都是在LuaStack中完成的。   
```
int LuaEngine::executeString(const char *codes)
{
    int ret = _stack->executeString(codes);
    _stack->clean();
    return ret;
}

int LuaEngine::executeScriptFile(const char* filename)
{
    int ret = _stack->executeScriptFile(filename);
    _stack->clean();
    return ret;
}

int LuaEngine::executeGlobalFunction(const char* functionName)
{
    int ret = _stack->executeGlobalFunction(functionName);
    _stack->clean();
    return ret;
}
```   
可以看到LuaEngine都是通调用LuaStack方法来实现的   
```
int LuaStack::executeString(const char *codes)
{
    luaL_loadstring(_state, codes);
    return executeFunction(0);
}

int LuaStack::executeScriptFile(const char* filename)
{
    std::string code("require \"");
    code.append(filename);
    code.append("\"");
    return executeString(code.c_str());
}

int LuaStack::executeGlobalFunction(const char* functionName)
{
    lua_getglobal(_state, functionName);       /* query function by name, stack: function */
    if (!lua_isfunction(_state, -1))
    {
        CCLOG("[LUA ERROR] name '%s' does not represent a Lua function", functionName);
        lua_pop(_state, 1);
        return 0;
    }
    return executeFunction(0);
}
```   
`executeString`和`executeScriptFile`加载代码到栈顶，然后调用`executeFunction`函数，`executeGlobalFunction`将函数名压入栈顶，然后检查是否是是函数，如果是也执行`executeFunction`函数记者来看这个重要的函数：   
```
int LuaStack::executeFunction(int numArgs)
{
    int functionIndex = -(numArgs + 1);
    if (!lua_isfunction(_state, functionIndex))
    {
        CCLOG("value at stack [%d] is not function", functionIndex);
        lua_pop(_state, numArgs + 1); // remove function and arguments
        return 0;
    }

    int traceback = 0;
    lua_getglobal(_state, "__G__TRACKBACK__");                         /* L: ... func arg1 arg2 ... G */
    if (!lua_isfunction(_state, -1))
    {
        lua_pop(_state, 1);                                            /* L: ... func arg1 arg2 ... */
    }
    else
    {
        lua_insert(_state, functionIndex - 1);                         /* L: ... G func arg1 arg2 ... */
        traceback = functionIndex - 1;
    }
    
    int error = 0;
    ++_callFromLua;
    error = lua_pcall(_state, numArgs, 1, traceback);                  /* L: ... [G] ret */
    --_callFromLua;
    if (error)
    {
        if (traceback == 0)
        {
            CCLOG("[LUA ERROR] %s", lua_tostring(_state, - 1));        /* L: ... error */
            lua_pop(_state, 1); // remove error message from stack
        }
        else                                                            /* L: ... G error */
        {
            lua_pop(_state, 2); // remove __G__TRACKBACK__ and error message from stack
        }
        return 0;
    }
    
    // get return value
    int ret = 0;
    if (lua_isnumber(_state, -1))
    {
        ret = (int)lua_tointeger(_state, -1);
    }
    else if (lua_isboolean(_state, -1))
    {
        ret = (int)lua_toboolean(_state, -1);
    }
    // remove return value from stack
    lua_pop(_state, 1);                                                /* L: ... [G] */
    
    if (traceback)
    {
        lua_pop(_state, 1); // remove __G__TRACKBACK__ from stack      /* L: ... */
    }
    
    return ret;
}
```   

1. 先判断栈顶元素是不是函数   
2. 先将`__G__TRACKBACK__`入栈，检查lua是否定义了此函数，如果没有定义，将`__G__TRACKBACK__`出战，如果定义了，将此函数插入调用函数之前，来跟踪调用顺序。   
3. 调用`lua_pcall`函数调用lua代码，如果调用函数出栈，将结果压入栈中，如果调用错误，将错误信息压入栈中。
4. 对调用结果进行处理    

这样就完成了cocos2d-x的c++到lua代码的调用。




