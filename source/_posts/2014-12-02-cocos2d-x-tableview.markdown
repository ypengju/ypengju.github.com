---
layout: post
title: "cocos2d-x -- TableView"
date: 2014-12-02 15:15:21 +0800
comments: true
categories: cocos2d-x
---
TableView是在游戏开发中最常使用的之一，它是ScrollView的子类，采用委托代理，只将数据交给用户处理，具体布局，内存处理都有引擎包装完成，对于加载大量列表数据非常方便，使得我们只需考虑，每个条目数据处理。   
TableView位于extensions扩展包中，所以在使用的时候需要引入头文件。   

```c++
#include "cocos-ext.h"
using namespace cocos2d::extension;
```   
在使用TableView的类中，需要继承`TableViewDataSource`，`TableViewDelegate`两个类。这两个类中定义了一些纯虚代理方法，需要在类中继承实现，还有其他的几个方法，可以选择实现。  <!--more-->

```c++ TableViewDelegate
class TableViewDelegate : public ScrollViewDelegate
{
public:
    //点击时调用
    virtual void tableCellTouched(TableView* table, TableViewCell* cell) = 0;

    //cell点下时调用
    virtual void tableCellHighlight(TableView* table, TableViewCell* cell){};

    //cell点下释放时调用
    virtual void tableCellUnhighlight(TableView* table, TableViewCell* cell){};

    //cell回收，在当前cell移除屏幕时调用，或者在reloadData()调用时，调用
    virtual void tableCellWillRecycle(TableView* table, TableViewCell* cell){};

};
```   
TableViewDelegate继承自ScrollViewDelegate，除了上边的这些方法外，还可实现ScrollViewDelegate中的`scrollViewDidScroll`和`scrollViewDidZoom`代理方法。  

```c++ TableViewDataSource
class TableViewDataSource
{
public:
    
    virtual ~TableViewDataSource() {}

    //每个cell的大小，用于指定tableView中每个Cell的大小
    virtual Size tableCellSizeForIndex(TableView *table, ssize_t idx) {
        return cellSizeForTable(table);
    };
    
    //默认cell的大小
    virtual Size cellSizeForTable(TableView *table) {
        return Size::ZERO;
    };
    
    //索引index处的TableViewCell内容
    virtual TableViewCell* tableCellAtIndex(TableView *table, ssize_t idx) = 0;
    
    //TableView中cell的个数
    virtual ssize_t numberOfCellsInTableView(TableView *table) = 0;
};
```   
在使用时，必须实现上边两个类中的纯虚函数，这是布局时必须的一些数据。在TableView中每个cell我们会用到`TableViewCell`这个类，这个只有一个简单的成员变量_idx用来标记cell的索引，但是它继承子Node，每个cell中用到的控件元素都会添加到这个类实例中进行管理。   
下边一个简单例子说明TableView的使用   
```
bool HelloWorld::init()
{
    if ( !Layer::init() )
    {
        return false;
    }
    
    Size winSize = Director::getInstance()->getWinSize();
    auto colorLayer = LayerColor::create(Color4B(125, 125, 125, 255));
    addChild(colorLayer);
    
    _tableView = TableView::create(this, Size(100, 500)); //指明此TableView的数据代理和可视区域大小
    _tableView->setPosition(Point(winSize.width/2, 0));
    _tableView->setDelegate(this); //指定代理为当前类，调用时调用我们申明的函数
    //设置滑动方向，只有两个方向，垂直滑动和水平滑动，默认垂直滑动，设置方向必须在调用reloadData()函数之前。
    _tableView->setDirection(ScrollView::Direction::HORIZONTAL); 
    //设置TableView中元素的排序方式，更具索引有从大到小，或者从小到大的顺序,这里值得关注下，在此方法中会调用reloadData()进行加载数据
    _tableView->setVerticalFillOrder(TableView::VerticalFillOrder::TOP_DOWN); 
    _tableView->setBounceable(false); //设置滑动是否有惯性
    this->addChild(_tableView);
    return true;
}

//设置每个cell的内容
TableViewCell* HelloWorld::tableCellAtIndex(TableView *table, ssize_t idx)
{
    CCLOG("-----------%ld", idx);
    TableViewCell *cell = table->dequeueCell(); //对移除屏幕的cell进行重复利用，避免额外开销

    if (!cell) {
        cell = TableViewCell::create();
        auto sp = Sprite::create("CloseNormal.png");
        sp->setPosition(Point(0, 0));
        sp->setAnchorPoint(Point(0, 0));
        sp->setTag(100);
        cell->addChild(sp);
        
        auto label = LabelTTF::create(StringUtils::format("%d", (int)idx).c_str(), "Arial", 20);
        label->setPosition(Point(sp->getContentSize().width/2, sp->getContentSize().height/2));
        label->setTag(200);
        cell->addChild(label);
    } else {
        auto label = (LabelTTF *)cell->getChildByTag(200);
        label->setString(StringUtils::format("%d", (int)idx).c_str());
    }
    return cell;
}
//TableView中的数量
ssize_t HelloWorld::numberOfCellsInTableView(TableView *table)
{
    return 100;
}
//每个cell的大小
cocos2d::Size HelloWorld::tableCellSizeForIndex(TableView *table, ssize_t idx)
{
    return Size(40, 40);
}
//cell点击调用
void HelloWorld::tableCellTouched(TableView* table, TableViewCell* cell)
{
    CCLOG("--------you touch me!!! %ld", cell->getIdx());
}
```   
实现这以上几个函数，既可以构建一个简单TableView，在调用过程中数据发生改变，我们只需调用TableView的reloadData()函数就可对数据进行刷新。还有其他几个函数可对TableView中指定索引元素进行更新，插入和删除操作。具体可见CCTableView.h中定义的接口。   

在lua中使用时，在定义相应函数后，我们需要对其进行注册，具体如下
```lua
tableView:registerScriptHandler(TableViewTestLayer.numberOfCellsInTableView,cc.NUMBER_OF_CELLS_IN_TABLEVIEW)  
tableView:registerScriptHandler(TableViewTestLayer.scrollViewDidScroll,cc.SCROLLVIEW_SCRIPT_SCROLL)
tableView:registerScriptHandler(TableViewTestLayer.scrollViewDidZoom,cc.SCROLLVIEW_SCRIPT_ZOOM)
tableView:registerScriptHandler(TableViewTestLayer.tableCellTouched,cc.TABLECELL_TOUCHED)
tableView:registerScriptHandler(TableViewTestLayer.cellSizeForTable,cc.TABLECELL_SIZE_FOR_INDEX)
tableView:registerScriptHandler(TableViewTestLayer.tableCellAtIndex,cc.TABLECELL_SIZE_AT_INDEX)
```   
这些具体函数必须在reloadData()函数之前注册，因为在reloadData()需要对数据进行处理。具体使用可参考官方的例子程序。
