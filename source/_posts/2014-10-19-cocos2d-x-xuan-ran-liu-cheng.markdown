---
layout: post
title: "cocos2d-x -- 渲染流程"
date: 2014-10-19 20:52:49 +0800
comments: true
categories: cocos2d-x
---
因为现在并不了解OpenGL ES的渲染过程，这里只能简单理清整个cocos2d-x 3.2版本的渲染顺序及调用过程，具体OpenGL ES实现，之后会学习总结。   
在3.0之前的版本中，cocos2d-x渲染每个节点都是放在节点的draw()方法中直接调用OpenGL命令来进行渲染的，比如2.2.5中的CCSprite::draw()<!-- more -->   
```c++ 
void CCSprite::draw(void)
{
    CC_PROFILER_START_CATEGORY(kCCProfilerCategorySprite, "CCSprite - draw");

    CCAssert(!m_pobBatchNode, "If CCSprite is being rendered by CCSpriteBatchNode, CCSprite#draw SHOULD NOT be called");

    CC_NODE_DRAW_SETUP();

    ccGLBlendFunc( m_sBlendFunc.src, m_sBlendFunc.dst );

    ccGLBindTexture2D( m_pobTexture->getName() );
    ccGLEnableVertexAttribs( kCCVertexAttribFlag_PosColorTex );

#define kQuadSize sizeof(m_sQuad.bl)
#ifdef EMSCRIPTEN
    long offset = 0;
    setGLBufferData(&m_sQuad, 4 * kQuadSize, 0);
#else
    long offset = (long)&m_sQuad;
#endif // EMSCRIPTEN

    // vertex
    int diff = offsetof( ccV3F_C4B_T2F, vertices);
    glVertexAttribPointer(kCCVertexAttrib_Position, 3, GL_FLOAT, GL_FALSE, kQuadSize, (void*) (offset + diff));

    // texCoods
    diff = offsetof( ccV3F_C4B_T2F, texCoords);
    glVertexAttribPointer(kCCVertexAttrib_TexCoords, 2, GL_FLOAT, GL_FALSE, kQuadSize, (void*)(offset + diff));

    // color
    diff = offsetof( ccV3F_C4B_T2F, colors);
    glVertexAttribPointer(kCCVertexAttrib_Color, 4, GL_UNSIGNED_BYTE, GL_TRUE, kQuadSize, (void*)(offset + diff));


    glDrawArrays(GL_TRIANGLE_STRIP, 0, 4);

    CHECK_GL_ERROR_DEBUG();


#if CC_SPRITE_DEBUG_DRAW == 1
    // draw bounding box
    CCPoint vertices[4]={
        ccp(m_sQuad.tl.vertices.x,m_sQuad.tl.vertices.y),
        ccp(m_sQuad.bl.vertices.x,m_sQuad.bl.vertices.y),
        ccp(m_sQuad.br.vertices.x,m_sQuad.br.vertices.y),
        ccp(m_sQuad.tr.vertices.x,m_sQuad.tr.vertices.y),
    };
    ccDrawPoly(vertices, 4, true);
#elif CC_SPRITE_DEBUG_DRAW == 2
    // draw texture box
    CCSize s = this->getTextureRect().size;
    CCPoint offsetPix = this->getOffsetPosition();
    CCPoint vertices[4] = {
        ccp(offsetPix.x,offsetPix.y), ccp(offsetPix.x+s.width,offsetPix.y),
        ccp(offsetPix.x+s.width,offsetPix.y+s.height), ccp(offsetPix.x,offsetPix.y+s.height)
    };
    ccDrawPoly(vertices, 4, true);
#endif // CC_SPRITE_DEBUG_DRAW

    CC_INCREMENT_GL_DRAWS(1);

    CC_PROFILER_STOP_CATEGORY(kCCProfilerCategorySprite, "CCSprite - draw");
}
```   
可以看到其中是直接调用OpenGL命令glXXX等直接进行渲染的，但是在3.0之后的版本，cocos2d-x改变了渲染策略，这里主要记录3.2版本的渲染方式。       
每一帧的渲染都是由主循环开始   
```
void DisplayLinkDirector::mainLoop()
{
    if (_purgeDirectorInNextLoop)
    {
        _purgeDirectorInNextLoop = false;
        purgeDirector();
    }
    else if (! _invalid)
    {
    	 //渲染场景
        drawScene();
     
        // release the objects
        PoolManager::getInstance()->getCurrentPool()->clear();
    }
}
```   
其中每一帧都调用drawScene()方法，对当前场景进行渲染。   
```
// Draw the Scene
void Director::drawScene()
{
    // calculate "global" dt
    calculateDeltaTime();
    
    // skip one flame when _deltaTime equal to zero.
    if(_deltaTime < FLT_EPSILON)
    {
        return;
    }

    if (_openGLView)
    {
        _openGLView->pollInputEvents();
    }

    //tick before glClear: issue #533
    if (! _paused)
    {
        _scheduler->update(_deltaTime);
        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
    }

    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    /* to avoid flickr, nextScene MUST be here: after tick and before draw.
     XXX: Which bug is this one. It seems that it can't be reproduced with v0.9 */
    if (_nextScene)
    {
        setNextScene();
    }

    pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    // draw the scene
    if (_runningScene)
    {
    	//调用当前场景的visit函数
        _runningScene->visit(_renderer, Mat4::IDENTITY, false);
        _eventDispatcher->dispatchEvent(_eventAfterVisit);
    }

    // draw the notifications node
    if (_notificationNode)
    {
        _notificationNode->visit(_renderer, Mat4::IDENTITY, false);
    }

    if (_displayStats)
    {
        showStats();
    }

	//进行渲染
    _renderer->render();
    _eventDispatcher->dispatchEvent(_eventAfterDraw);

    popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);

    _totalFrames++;

    // swap buffers
    if (_openGLView)
    {
        _openGLView->swapBuffers();
    }

    if (_displayStats)
    {
        calculateMPF();
    }
}
```   
可以看到这里先调用了当前场景的visit函数，之后进行渲染，那先来看看当前场景的vivit函数。在Scene中并未实现visit函数，所以调用的是父类Node的visit函数。   
```
void Node::visit(Renderer* renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    // quick return if not visible. children won't be drawn.
    if (!_visible)
    {
        return;
    }

    uint32_t flags = processParentFlags(parentTransform, parentFlags);

    // IMPORTANT:
    // To ease the migration to v3.0, we still support the Mat4 stack,
    // but it is deprecated and your code should not rely on it
    Director* director = Director::getInstance();
    director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _modelViewTransform);

    int i = 0;

    if(!_children.empty())
    {
        sortAllChildren();
        // draw children zOrder < 0
        for( ; i < _children.size(); i++ )
        {
            auto node = _children.at(i);

            if ( node && node->_localZOrder < 0 )
                node->visit(renderer, _modelViewTransform, flags);
            else
                break;
        }
        // self draw
        this->draw(renderer, _modelViewTransform, flags);

        for(auto it=_children.cbegin()+i; it != _children.cend(); ++it)
            (*it)->visit(renderer, _modelViewTransform, flags);
    }
    else
    {
        this->draw(renderer, _modelViewTransform, flags);
    }

    director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    
    // FIX ME: Why need to set _orderOfArrival to 0??
    // Please refer to https://github.com/cocos2d/cocos2d-x/pull/6920
    // reset for next frame
    // _orderOfArrival = 0;
}
```   
可以看到这里递归调用了所以子节点的visit函数，在visit函数中会调用所有节点的draw函数，在3.0之前的版本中draw函数具体实现OpenGL渲染，在3.0之后的版本中，draw函数中，向引擎发送了一个渲染命了。   
```
void Node::draw(Renderer* renderer, const Mat4 &transform, uint32_t flags)
{
}
```   
Node的中的draw函数为虚函数，没有实现。以Sprite为例来看看具体流程。   
```
void Sprite::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    // Don't do calculate the culling if the transform was not updated
    _insideBounds = (flags & FLAGS_TRANSFORM_DIRTY) ? renderer->checkVisibility(transform, _contentSize) : _insideBounds;

    if(_insideBounds)
    {
        _quadCommand.init(_globalZOrder, _texture->getName(), getGLProgramState(), _blendFunc, &_quad, 1, transform);
        renderer->addCommand(&_quadCommand);
#if CC_SPRITE_DEBUG_DRAW
        _customDebugDrawCommand.init(_globalZOrder);
        _customDebugDrawCommand.func = CC_CALLBACK_0(Sprite::drawDebugData, this);
        renderer->addCommand(&_customDebugDrawCommand);
#endif //CC_SPRITE_DEBUG_DRAW
    }
}
```
可以看到，Sprite的draw函数中，并没有进行具体的OpenGL操作，而是在renderer中添加了而一个渲染命令。而真正调用OpenGL命令进行渲染是在Renderer的render函数中。   
```
void Renderer::render()
{
    //Uncomment this once everything is rendered by new renderer
    //glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

    //TODO setup camera or MVP
    _isRendering = true;
    
    if (_glViewAssigned)
    {
        // cleanup
        _drawnBatches = _drawnVertices = 0;

        //Process render commands
        //1. Sort render commands based on ID
        for (auto &renderqueue : _renderGroups)
        {
            renderqueue.sort();
        }
        visitRenderQueue(_renderGroups[0]);
        flush();
    }
    clean();
    _isRendering = false;
}
```   
render函数中先对renderqueue进行了排序，renderqueue中存储这所有在节点draw函数加载进来的渲染命令。排序完之后，调用visitRenderQueue函数完成具体的渲染工作。   
```
void Renderer::visitRenderQueue(const RenderQueue& queue)
{
    ssize_t size = queue.size();
    
    for (ssize_t index = 0; index < size; ++index)
    {
        auto command = queue[index];
        auto commandType = command->getType();
        if(RenderCommand::Type::QUAD_COMMAND == commandType)
        {
            flush3D();
            auto cmd = static_cast<QuadCommand*>(command);
            //Batch quads
            if(_numQuads + cmd->getQuadCount() > VBO_SIZE)
            {
                CCASSERT(cmd->getQuadCount()>= 0 && cmd->getQuadCount() < VBO_SIZE, "VBO is not big enough for quad data, please break the quad data down or use customized render command");
                
                //Draw batched quads if VBO is full
                drawBatchedQuads();
            }
            
            _batchedQuadCommands.push_back(cmd);
            
            memcpy(_quads + _numQuads, cmd->getQuads(), sizeof(V3F_C4B_T2F_Quad) * cmd->getQuadCount());
            convertToWorldCoordinates(_quads + _numQuads, cmd->getQuadCount(), cmd->getModelView());
            
            _numQuads += cmd->getQuadCount();

        }
        else if(RenderCommand::Type::GROUP_COMMAND == commandType)
        {
            flush();
            int renderQueueID = ((GroupCommand*) command)->getRenderQueueID();
            visitRenderQueue(_renderGroups[renderQueueID]);
        }
        else if(RenderCommand::Type::CUSTOM_COMMAND == commandType)
        {
            flush();
            auto cmd = static_cast<CustomCommand*>(command);
            cmd->execute();
        }
        else if(RenderCommand::Type::BATCH_COMMAND == commandType)
        {
            flush();
            auto cmd = static_cast<BatchCommand*>(command);
            cmd->execute();
        }
        else if (RenderCommand::Type::MESH_COMMAND == commandType)
        {
            flush2D();
            auto cmd = static_cast<MeshCommand*>(command);
            if (_lastBatchedMeshCommand == nullptr || _lastBatchedMeshCommand->getMaterialID() != cmd->getMaterialID())
            {
                flush3D();
                cmd->preBatchDraw();
                cmd->batchDraw();
                _lastBatchedMeshCommand = cmd;
            }
            else
            {
                cmd->batchDraw();
            }
        }
        else
        {
            CCLOGERROR("Unknown commands in renderQueue");
        }
    }
}
```   
可以看到针对不同渲染命令，会进行具体渲染操作。   
这里只简单整理了整个渲染流程的，理清思路，对具体操作并未深入研究，之后会研究渲染细节操作，及不同渲染命令分析。   
###参考

* [Cocos2d-x 3.2与OpenGL渲染总结（一）：Cocos2d-x 3.2的渲染流程](http://blog.csdn.net/cbbbc/article/details/39449945)
* [从Cocos2d-x学习OpenGL -- Cocos2d-x渲染结构](http://cn.cocos2d-x.org/tutorial/show?id=1025)


