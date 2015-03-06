---
layout: post
title: "cocos2d-x -- 富文本RichText"
date: 2015-03-05 14:45:11 +0800
comments: true
categories: cocos2d-x
---
cocos2d-x在3.X版本中增加了富文本控件，使得处理游戏中多颜色字体及图片方便很多，这篇文章就学习总结下富文本控件RichText。RichText继承自Widget类，每个具体节点由RichElement的子类RichElementText，RichElementImage，RichElementCustomNode来处理文字，图片和自定义节点。当我们生成好RichElement的子类时，将其添加到RichText中，来进行富文本显示，像下边这样显示两种颜色文字。
```cpp
auto rich = RichText::create();
rich->setPosition(Vec2(visibleSize.width/2, visibleSize.height/2));
this->addChild(rich);

auto element1 = RichElementText::create(100, Color3B(0, 255, 123), 125, "Hello world", "Arial", 20);
rich->pushBackElement(element1);

auto element2 = RichElementText::create(101, Color3B(255, 0, 0), 255, "Hello cocos2d-x", "Arial", 20);
rich->pushBackElement(element2);
```
先来看看RichElement和它的几个子类。<!--more-->
```cpp RichElement
class CC_GUI_DLL RichElement : public Ref
{
public:
    enum class Type
    {
        TEXT,
        IMAGE,
        CUSTOM
    };
    RichElement(){};
    virtual ~RichElement(){};
    bool init(int tag, const Color3B& color, GLubyte opacity);
protected:
    Type _type;
    int _tag;
    Color3B _color;
    GLubyte _opacity;
    friend class RichText;
};
```
RichElement是父类，记录了子类一些公有的属性，类型，标签，颜色，透明度等。   
RichElementText添加了字体文本，字体，字体大小属性。    
RichElementImage添加了图片路径属性。   
RichElementCustomNode添加了一个Node节点，自定义的一些内容，都存储在这个节点中。   
其实单独看这几个类也没干什么，只是包装了一些信息，而具体的操作，都在RichText类中。   
```cpp RichText
class CC_GUI_DLL RichText : public Widget
{
public:
    RichText();
    virtual ~RichText();
    static RichText* create();
    //对Vector进行操作
    void insertElement(RichElement* element, int index);
    void pushBackElement(RichElement* element);
    void removeElement(int index);
    void removeElement(RichElement* element);
    //设置行间距
    void setVerticalSpace(float space);
    //设置锚点
    virtual void setAnchorPoint(const Vec2 &pt);
    //获取content大小
    virtual Size getVirtualRendererSize() const override;
    //格式化文本
    void formatText();
    //设置为false时，setContentSize才有效
    virtual void ignoreContentAdaptWithSize(bool ignore);
    //获取描述
    virtual std::string getDescription() const override;
    
CC_CONSTRUCTOR_ACCESS:
    virtual bool init() override;
    
protected:
    virtual void adaptRenderers();

    virtual void initRenderer();
    void pushToContainer(Node* renderer);
    void handleTextRenderer(const std::string& text, const std::string& fontName, float fontSize, const Color3B& color, GLubyte opacity);
    void handleImageRenderer(const std::string& fileParh, const Color3B& color, GLubyte opacity);
    void handleCustomRenderer(Node* renderer);
    void formarRenderers();
    void addNewLine();
protected:
    //标记是否格式化文本
    bool _formatTextDirty;
    //存储RichElement的容器，所有RichText添加进来的RichElement都存储在该容器中
    Vector<RichElement*> _richElements;
    //_elementRenders中的每个Vector<Node*>中的节点，是每行的所有节点
    std::vector<Vector<Node*>*> _elementRenders;
    //每行剩余宽度，在需要换行时使用
    float _leftSpaceWidth;
    //行间距
    float _verticalSpace;
    //代表Rich的节点，所有的RichElement都添加在该节点下
    Node* _elementRenderersContainer;
};
```
首先来看添加删除的几个方法
```
void RichText::insertElement(RichElement *element, int index)
{
    _richElements.insert(index, element);
    _formatTextDirty = true;
}
    
void RichText::pushBackElement(RichElement *element)
{
    _richElements.pushBack(element);
    _formatTextDirty = true;
}
    
void RichText::removeElement(int index)
{
    _richElements.erase(index);
    _formatTextDirty = true;
}
    
void RichText::removeElement(RichElement *element)
{
    _richElements.eraseObject(element);
    _formatTextDirty = true;
}
```
看这几个方法就会发现，添加和删除其实都是在操作_richElements容器，并标记需要格式化文本。    
拿来看看添加后格式化文本干了什么   
```cpp formatText()
void RichText::formatText()
{
    //判断是否需要格式化文本
    if (_formatTextDirty)
    {
        //清除_elementRenderersContainer节点下的所有子节点
        _elementRenderersContainer->removeAllChildren();
        _elementRenders.clear();
        //判断是否忽略content大小
        //默认为true，此时调用setContentSize()无效，文本不会换行，否则会根据设置宽度对文本换行
        if (_ignoreSize)
        {
            //此时不进行距离计算，遍历_richElements，根据类型还原RichElement子类，
            //调用pushToContainer()添加到_elementRenders渲染容器中。
            addNewLine();
            for (ssize_t i=0; i<_richElements.size(); i++)
            {
                RichElement* element = _richElements.at(i);
                Node* elementRenderer = nullptr;
                switch (element->_type)
                {
                    case RichElement::Type::TEXT:
                    {
                        RichElementText* elmtText = static_cast<RichElementText*>(element);
                        if (FileUtils::getInstance()->isFileExist(elmtText->_fontName))
                        {
                            elementRenderer = Label::createWithTTF(elmtText->_text.c_str(), elmtText->_fontName, elmtText->_fontSize);
                        }
                        else
                        {
                            elementRenderer = Label::createWithSystemFont(elmtText->_text.c_str(), elmtText->_fontName, elmtText->_fontSize);
                        }
                        break;
                    }
                    case RichElement::Type::IMAGE:
                    {
                        RichElementImage* elmtImage = static_cast<RichElementImage*>(element);
                        elementRenderer = Sprite::create(elmtImage->_filePath.c_str());
                        break;
                    }
                    case RichElement::Type::CUSTOM:
                    {
                        RichElementCustomNode* elmtCustom = static_cast<RichElementCustomNode*>(element);
                        elementRenderer = elmtCustom->_customNode;
                        break;
                    }
                    default:
                        break;
                }
                elementRenderer->setColor(element->_color);
                elementRenderer->setOpacity(element->_opacity);
                pushToContainer(elementRenderer);
            }
        }
        else
        {
            //因为要根据contentSize进行换行，所以要根据RichElement内容进行长度计算，调用对应handleXXXRenderer方法进行计算
            addNewLine();
            for (ssize_t i=0; i<_richElements.size(); i++)
            {
                RichElement* element = static_cast<RichElement*>(_richElements.at(i));
                switch (element->_type)
                {
                    case RichElement::Type::TEXT:
                    {
                        RichElementText* elmtText = static_cast<RichElementText*>(element);
                        handleTextRenderer(elmtText->_text.c_str(), elmtText->_fontName.c_str(), elmtText->_fontSize, elmtText->_color, elmtText->_opacity);
                        break;
                    }
                    case RichElement::Type::IMAGE:
                    {
                        RichElementImage* elmtImage = static_cast<RichElementImage*>(element);
                        handleImageRenderer(elmtImage->_filePath.c_str(), elmtImage->_color, elmtImage->_opacity);
                        break;
                    }
                    case RichElement::Type::CUSTOM:
                    {
                        RichElementCustomNode* elmtCustom = static_cast<RichElementCustomNode*>(element);
                        handleCustomRenderer(elmtCustom->_customNode);
                        break;
                    }
                    default:
                        break;
                }
            }
        }
        formarRenderers();
        _formatTextDirty = false;
    }
}
```
formatText()主要对_richElements容器内的元素进行分类，text其实就是一个Label,image是Sprite,而custom就是一个Node节点。忽略content大小的比较简单，直接添加节点就好了，而计算的就比较麻烦了，可以看看对text的计算方法，image和custom类似。
```cpp handleTextRenderer
void RichText::handleTextRenderer(const std::string& text, const std::string& fontName, float fontSize, const Color3B &color, GLubyte opacity)
{
    //先判断是否存在字体文件，然后调用不同的Label方法
    auto fileExist = FileUtils::getInstance()->isFileExist(fontName);
    Label* textRenderer = nullptr;
    if (fileExist)
    {
        textRenderer = Label::createWithTTF(text, fontName, fontSize);
    } 
    else
    {
        textRenderer = Label::createWithSystemFont(text, fontName, fontSize);
    }
    //获取文本宽度
    float textRendererWidth = textRenderer->getContentSize().width;
    //计算一行剩下的宽度
    _leftSpaceWidth -= textRendererWidth;
    //如果剩余宽度小于0，就需要对文本进行换行，创建新的Label
    if (_leftSpaceWidth < 0.0f)
    {
        //计算超出文本的比例
        float overstepPercent = (-_leftSpaceWidth) / textRendererWidth;
        std::string curText = text;
        //计算原始文本长度
        size_t stringLength = StringUtils::getCharacterCountInUTF8String(text);
        //计算未超出长度
        int leftLength = stringLength * (1.0f - overstepPercent);
        //未超出文本
        std::string leftWords = Helper::getSubStringOfUTF8String(curText,0,leftLength);
        //超出文本
        std::string cutWords = Helper::getSubStringOfUTF8String(curText, leftLength, stringLength - leftLength);
        //对未查出文本生成的Label
        if (leftLength > 0)
        {
            Label* leftRenderer = nullptr;
            if (fileExist)
            {
                leftRenderer = Label::createWithTTF(Helper::getSubStringOfUTF8String(leftWords, 0, leftLength), fontName, fontSize);
            }
            else
            {
                leftRenderer = Label::createWithSystemFont(Helper::getSubStringOfUTF8String(leftWords, 0, leftLength), fontName, fontSize);
            }
            if (leftRenderer)
            {
                leftRenderer->setColor(color);
                leftRenderer->setOpacity(opacity);
                pushToContainer(leftRenderer);
            }
        }
        //换行，递归调用handleTextRenderer()对超出文本进行计算
        addNewLine();
        handleTextRenderer(cutWords.c_str(), fontName, fontSize, color, opacity);
    }
    else
    {
        textRenderer->setColor(color);
        textRenderer->setOpacity(opacity);
        pushToContainer(textRenderer);
    }
}
```
上边方法处理了RichText文本宽度问题，那高度问题是怎么处理的，那就得看formarRenderers方法了
```cpp formarRenderers
void RichText::formarRenderers()
{
    if (_ignoreSize)
    {
        //不需要换行的，以所有元素中高度最高的为_elementRenderersContainer节点的高度
        float newContentSizeWidth = 0.0f;
        float newContentSizeHeight = 0.0f;
        
        Vector<Node*>* row = (_elementRenders[0]);
        float nextPosX = 0.0f;
        for (ssize_t j=0; j<row->size(); j++)
        {
            Node* l = row->at(j);
            l->setAnchorPoint(Vec2::ZERO);
            l->setPosition(nextPosX, 0.0f);
            _elementRenderersContainer->addChild(l, 1);
            Size iSize = l->getContentSize();
            newContentSizeWidth += iSize.width;
            newContentSizeHeight = MAX(newContentSizeHeight, iSize.height);
            nextPosX += iSize.width;
        }
        _elementRenderersContainer->setContentSize(Size(newContentSizeWidth, newContentSizeHeight));
    }
    else
    {
        //需要换行的，需要计算每行最高的外，还需要加上行间距
        float newContentSizeHeight = 0.0f;
        float *maxHeights = new float[_elementRenders.size()];
        
        for (size_t i=0; i<_elementRenders.size(); i++)
        {
            Vector<Node*>* row = (_elementRenders[i]);
            float maxHeight = 0.0f;
            for (ssize_t j=0; j<row->size(); j++)
            {
                Node* l = row->at(j);
                maxHeight = MAX(l->getContentSize().height, maxHeight);
            }
            maxHeights[i] = maxHeight;
            newContentSizeHeight += maxHeights[i];
        }
        
        
        float nextPosY = _customSize.height;
        for (size_t i=0; i<_elementRenders.size(); i++)
        {
            Vector<Node*>* row = (_elementRenders[i]);
            float nextPosX = 0.0f;
            nextPosY -= (maxHeights[i] + _verticalSpace);
            
            for (ssize_t j=0; j<row->size(); j++)
            {
                Node* l = row->at(j);
                l->setAnchorPoint(Vec2::ZERO);
                l->setPosition(nextPosX, nextPosY);
                _elementRenderersContainer->addChild(l, 1);
                nextPosX += l->getContentSize().width;
            }
        }
        _elementRenderersContainer->setContentSize(_contentSize);
        delete [] maxHeights;
    }
    //清除_elementRenders容器元素
    size_t length = _elementRenders.size();
    for (size_t i = 0; i<length; i++)
    {
        Vector<Node*>* l = _elementRenders[i];
        l->clear();
        delete l;
    }    
    _elementRenders.clear();
    
    if (_ignoreSize)
    {
        Size s = getVirtualRendererSize();
        this->setContentSize(s);
    }
    else
    {
        this->setContentSize(_customSize);
    }
    updateContentSizeWithTextureSize(_contentSize);
    _elementRenderersContainer->setPosition(_contentSize.width / 2.0f, _contentSize.height / 2.0f);
}
```   
以上就是RichText富文本处理过程了。
