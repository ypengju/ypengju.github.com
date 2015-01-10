---
layout: post
title: "cocos2d-x -- Http请求"
date: 2014-11-07 21:40:47 +0800
comments: true
categories: cocos2d-x
---
cocos2d-x的网络请求相关代码是放在network目录下的，所以在使用的时候，首先要导入头文件。   
newwork目录下主要由这几个类  

* HttpRequest   
* HttpResponse   
* HttpClient   
* SocketIO   
* WebSocket   

这篇主要总结下Http请求的处理过程。   <!--more-->
首先请求之前，我们需要组织请求数据，这个就是由HttpRequest来完成的。   
比如设置请求地址，请求数据，请求类型，响应回调函数等。如下例   
```c++
cocos2d::network::HttpRequest *request = new cocos2d::network::HttpRequest();
request->setUrl("http://127.0.0.1:1337/");
request->setRequestType(cocos2d::network::HttpRequest::Type::GET);
request->setResponseCallback(CC_CALLBACK_2(HelloWorld::onHttpRequestCompleted,this));
request->setTag("get request”);
request->setRequestData(const char *buffer, size_t len);
cocos2d::network::HttpClient::getInstance()->send(request);
request->release();
```    
上边例子我们处理了一个请求内容，设置请求数据的时候，我们还需要知道他的大小。   
请求方式有四种   
```c++
enum class Type
{
    GET,
    POST,
    PUT,
    DELETE,
    UNKNOWN,
};
```   
有GET，POST，PUT, DELETE分别对应http请求类型，一般应该用得比较多的是GET请求和POST请求了。除此之外，我们还可以通过`void setHeaders(std::vector<std::string> pHeaders)`方法设置请求头信息。请求数据组织好了，我们就需要把它发送出去了，这个发送管理任务就由HttpClient来完成了，就像上边这句一样：
`cocos2d::network::HttpClient::getInstance()->send(request)`可以看到，他是一个单例。   
HttpClient用来管理请求的发送，设置连接超时时间，设置读取响应时间等，以及请求的线程管理等。HttpClient处理请求有两种方法，一种是send(),一种是sendImmediate()直接请求。   
先看看send()都干了什么   
```c++
void HttpClient::send(HttpRequest* request)
{    
    if (false == lazyInitThreadSemphore()) 
    {
        return;
    }
    
    if (!request)
    {
        return;
    }
        
    request->retain();
    
    if (nullptr != s_requestQueue) {
        s_requestQueueMutex.lock();
        s_requestQueue->pushBack(request);
        s_requestQueueMutex.unlock();
        
        // Notify thread start to work
        s_SleepCondition.notify_one();
    }
}
```    
首先调用了lazyInitThreadSemphore(),在这个方法中，初始花了请求与响应队列，新起了一个线程，用于请求响应工作。然后将请求数据，添加到请求响应队列中，然后通知线程工作。   
```c++
//Lazy create semaphore & mutex & thread
bool HttpClient::lazyInitThreadSemphore()
{
    if (s_requestQueue != nullptr) {
        return true;
    } else {
        
        s_requestQueue = new Vector<HttpRequest*>();
        s_responseQueue = new Vector<HttpResponse*>();
        
		s_need_quit = false;
		
        auto t = std::thread(CC_CALLBACK_0(HttpClient::networkThread, this));
        t.detach();
    }
    
    return true;
}
```   
主要工作在networkThread中完成  
```c++
void HttpClient::networkThread()
{    
    HttpRequest *request = nullptr;
    
    auto scheduler = Director::getInstance()->getScheduler();
    
    while (true) 
    {
        if (s_need_quit)
        {
            break;
        }
        
        // step 1: send http request if the requestQueue isn't empty
        request = nullptr;
        
        s_requestQueueMutex.lock();
        
        //Get request task from queue
        
        if (!s_requestQueue->empty())
        {
            request = s_requestQueue->at(0);
            s_requestQueue->erase(0);
        }
        
        s_requestQueueMutex.unlock();
        
        if (nullptr == request)
        {
            // Wait for http request tasks from main thread
            std::unique_lock<std::mutex> lk(s_SleepMutex); 
            s_SleepCondition.wait(lk);
            continue;
        }
        
        // step 2: libcurl sync access
        
        // Create a HttpResponse object, the default setting is http access failed
        HttpResponse *response = new HttpResponse(request);
        
        processResponse(response, s_errorBuffer);
        

        // add response packet into queue
        s_responseQueueMutex.lock();
        s_responseQueue->pushBack(response);
        s_responseQueueMutex.unlock();
        
        if (nullptr != s_pHttpClient) {
            scheduler->performFunctionInCocosThread(CC_CALLBACK_0(HttpClient::dispatchResponseCallbacks, this));
        }
    }
    
    // cleanup: if worker thread received quit signal, clean up un-completed request queue
    s_requestQueueMutex.lock();
    s_requestQueue->clear();
    s_requestQueueMutex.unlock();
    
    
    if (s_requestQueue != nullptr) {
        delete s_requestQueue;
        s_requestQueue = nullptr;
        delete s_responseQueue;
        s_responseQueue = nullptr;
    }
    
}
```   
整个处理流程是这样的:

* 先初始化请求和响应队列，生成新的线程，新线程中调用networkThread()函数    
* 在networkThread()函数中有一个while(true)循环，用来检查请求队列是否有新的请求数据，如果有，处理之。    
* 当在send()函数中，将请求数据加入请求队列后，networkThread()随后进行处理。    
* 在networkThread()中，先将请求数据从请求队列中拿出来，然后此数据生成HttpResponse()响应。    
* 在HttpResponse中保存有一个请求数据，然后调用processResponse()方法，进行响应处理。    
* 在processResponse()方法中，先将response中的请求数据拿出来，根据请求类型，调用不同的方法。    
* processGetTask(), processPostTask(), processPutTask(), processDeleteTask()    
* 分别用于处理GET,POST,PUT,DELETE请求。    
* 在这些方法中，使用curl进行具体网络请求，链接等。并将结果存储到response中。    
* 然后将response添加到响应队列中。用schedule调回主线程dispatchResponseCallbacks()方法进行处理。    
* 在dispatchResponseCallbacks()中，先将响应从响应队列中取出。    
* 得到response中的request请求，然后调用request中的注册的callback函数。    
* 此时就可以在callback函数中，通过response中的响应数据进行处理了。    

这样就完成了一次具体的http请求。   
相比sendImmediate()请求就简单直接一些：  

* 在新线程中，调用networkThreadAlone()方法。   
* 在networkThreadAlone()没有将request添加到队列中，而是直接生成response，采用processResponse()方法进行处理。   
* 处理之后，schedule调回主线程，调用callback，进行数据处理。   

以上就是对cocos2d-x的Http请求的简单学习总结。

