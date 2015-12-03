---
layout: post
title: "给CCScrollView添加滚动阻尼效果"
date:   2015-04-10
categories: Game Dev
---

最近我们在修改越南版本的UI，在新版本的UI上遇到两个一直未能解决问题。一是Home界面在滑动时很容易触摸到按钮，导致响应了按钮的事件。二是Home界面滑动太僵硬，没有流畅感。

Home界面是由一个屏幕大小的TableView组成，然后塞进一个超过屏幕大小的Cell（只塞进一个），在Cell上有很多按钮（位置与形状不规则），玩家可以自由滑动后点击按钮进入相应的界面。可以参考刀塔传奇的首页做法。

问题：在滑动过程中，玩家很容易触碰到按钮，导致响应了按钮事件，涉及到如何处理CCTableView与CCControlButton响应事件优先级问题。

CCTableView在滑动过程中会按照顺序触发下面几个函数：  

	virtual bool ccTouchBegan(cocos2d::CCTouch* pTouch, cocos2d::CCEvent* pEvent);
    virtual void ccTouchMoved(cocos2d::CCTouch* pTouch, cocos2d::CCEvent* pEvent);
	
	virtual void ccTouchEnded(cocos2d::CCTouch *pTouch, cocos2d::CCEvent *pEvent);
	virtual void ccTouchCancelled(cocos2d::CCTouch *pTouch, cocos2d::CCEvent *pEvent);

在滑动过程中还会出发一个函数：  

	virtual void scrollViewDidScroll(cocos2d::extension::CCScrollView* view);

此函数的触发有一个条件，就是只要CCTableView在滚动，这个函数就会被调用。解决方法：  

标记当前的CCTableView是否处于滚动状态，在```scrollViewDidScroll```或者```ccTouchMoved```中将滚动标记设置为true，在```ccTouchEnded```设置为false，同时将CCControlButton的事件响应优先级调低，即先于CCTableView响应触控事件，这样在玩家触控过程中，在按钮的响应时间里获得当前CCTableView的状态，如果是滚动状态则返回不做处理即可。

Home界面滑动僵硬的问题。由于Home界面是利用CCTableView里只塞一个Cell实现的，所以在滑动顶端或者底端的时候会回弹，策划说这种方式玩家感受不太好，要做成刀塔传奇那样，滑到顶端或者底端就不能再滑动了。关于这个问题，CCTableView有一个接口可以设置其滚动的风格：	  

	void setBounceable(bool bBounceable) 
	{ 
		m_bBounceable = bBounceable; 
	}

调用了这个接口之后，滚动到顶端或者底端确实不能再滚动了，但是带来了另外一个问题，就是滑动很僵硬，只要手指离开屏幕，滚动就会立即停止，没有滚动惯性。而且策划要求滚动到顶端或者底端后不能再滚动了而且要求滑动有惯性和流畅，类似iOS中的UITableView一样。明显要修改我们封装的XTableView的代码了，经过一番努力此问题成功解决，此问题可以拆分成两个问题：  
1. 滑动在顶端或者底端时不能再滑动了。  
2. 滑动到顶端或者底端时手指继续滑动也不能带动CCSrcollView滑动。  

实际上就是在滑动到底之后，需要有一个阻尼效果，这个阻尼足够大的时候就达到了不能在滑动的效果。对于第二个问题，可能会有疑惑。因为第一个问题的成立是Cell已经停留在了最底端，意味着Cell此时是停下来的。但是当如果快速的滑动时，当Cell还来不及停下来的时候，Cell会跟着滑动方向再滑动一段距离，这也是策划不能接受地方。

对于第一个问题，我们只需要在滑动时计算滑动距离偏移时成比例的减小即可。如何添加阻尼效果呢，我参照了cocos2d-x 3.x版本的代码，发现3.x版本是有阻尼效果的。在3.x版本中，阻尼效果是在```onTouchMoved```函数中计算的，在计算滚动偏移的时候，将目的坐标按照比例减小即可实现，宏```BOUNCE_BACK_FACTOR```设定了这个比例，默认是0.35f，3.x版本中这个值被写死了。于是我重写了XTableView中的```ccTouchMoved```函数实现了。

对于第二个问题，继续滑动的时候也是有变量经过计算的，奥义就在```CCScrollView::updateInset```里。只要将这里面的计算量设置到足够小的话就可以实现即使快速滑动(当Cell未停下来)时Cell再滑动的距离变得很小。综合这两个效果就可以在```m_bBounceable```为true的情况下，滑动有惯性且流程，而且Cell在滑动到底之后不会再朝滑动方向滑动一段距离了。我看了CCScrollView的代码，发现到底之后再滑动距离是利用宏```INSET_RATIO```计算的，最大滑动距离是m\_fMaxInset和m\_fMinInset。这个距离是CCScrollView在```setContentSize```的时候调用```updateInset```计算的，那么如何修改这个值呢？```updateInset```函数是非虚函数，但是```setContentSize```是虚函数，因此重写这个虚函数，自己再实现一个```updateInsetEx```函数，在```setContentSize```的时候调用之即可。代码如下：

	void XTableView::setContentSize(const CCSize & size)
	{
		if (CCScrollView::getContainer() != NULL)
		{
			CCScrollView::getContainer()->setContentSize(size);
			this->updateInsetEx();
		}
	}
	
	// 实现滚动回弹
	void XTableView::updateInsetEx()
	{
		if (this->getContainer() != NULL)
		{
			// m_fInset是滑动到底之后再滑动的计算量
			// m_fMaxInset 最大滑动距离偏移
			// m_fMinInset 最小滑动距离偏移
			m_fMaxInset = this->maxContainerOffset();
			m_fMaxInset = ccp(m_fMaxInset.x + m_tViewSize.width * m_fInset,
				m_fMaxInset.y + m_tViewSize.height * m_fInset);
			m_fMinInset = this->minContainerOffset();
			m_fMinInset = ccp(m_fMinInset.x - m_tViewSize.width * m_fInset,
				m_fMinInset.y - m_tViewSize.height * m_fInset);
		}
	}
	
	// 实现滚动阻尼效果
	void XTableView::ccTouchMoved(CCTouch* pTouch, CCEvent* pEvent)
	{
		if (!m_bLoadCompleted)
			return;
	
		if (!this->isVisible())
			return;
		
		// 获取滑动方向
		CCScrollViewDirection euCurDirection = this->getDirection();
		m_bIsScrolling = true;
		if (m_pTouches->containsObject(pTouch))
		{
			// 检测当前触控点是否在触控点列表里
			if (m_pTouches->count() == 1 && this->isDragging())
			{
				// 单点触控，多点触控暂时不做处理
				CCPoint moveDistance, newPoint;
				CCRect  frame;
				float newX, newY;
				
				// 计算新的坐标和滑动距离偏移
				frame = this->getViewRect();
				newPoint = this->convertTouchToNodeSpace(	\
					static_cast<CCTouch*>(m_pTouches->objectAtIndex(0)));
				moveDistance = newPoint - m_tTouchPoint;
	
				float dis = 0.0f;
				if (euCurDirection == kCCScrollViewDirectionVertical)
				{
					dis = moveDistance.y;
					float pos = m_pContainer->getPosition().y;
					if (!(minContainerOffset().y <= pos && 
						pos <= maxContainerOffset().y)) 
					{
						moveDistance.y *= m_fBounceBackFactor;
					}
				}
				else if (euCurDirection == kCCScrollViewDirectionHorizontal)
				{
					// 阻尼效果，计算加上阻尼效果之后距离偏移量
					dis = moveDistance.x;
					float pos = m_pContainer->getPosition().x;
					if (!(minContainerOffset().x <= pos && 
						pos <= maxContainerOffset().x)) 
					{
						moveDistance.x *= m_fBounceBackFactor;
					}
				}
				else
				{
					dis = sqrtf(moveDistance.x*moveDistance.x + \  
								moveDistance.y*moveDistance.y);
	
					float pos = m_pContainer->getPosition().y;
					if (!(minContainerOffset().y <= pos && 
						pos <= maxContainerOffset().y)) 
					{
						//m_fBounceBackFactor是阻尼值
						moveDistance.y *= m_fBounceBackFactor;
					}
					
					pos = m_pContainer->getPosition().x;
					if (!(minContainerOffset().x <= pos && 
						pos <= maxContainerOffset().x)) 
					{
						moveDistance.x *= m_fBounceBackFactor;
					}
				}
	
				if (!this->isTouchMoved() && 
					fabs(convertDistanceFromPointToInch(dis)) < (7.0f / 160.0f))
				{
					return;
				}
	
				if (!this->isTouchMoved())
				{
					moveDistance = CCPointZero;
				}
	
				m_tTouchPoint = newPoint;
				m_bTouchMoved = true;
	
				if (this->isDragging())
				{
					// 横向触控与竖直触控时准确化
					// 横向时y坐标不变，竖直时x坐标不变
					switch (euCurDirection)
					{
					case kCCScrollViewDirectionVertical:
						moveDistance = CCPoint(0.0f, moveDistance.y);
						break;
					case kCCScrollViewDirectionHorizontal:
						moveDistance = CCPoint(moveDistance.x, 0.0f);
						break;
					default:
						break;
					}
					
					// 计算并设定新的位置坐标
					newX = m_pContainer->getPosition().x + moveDistance.x;
					newY = m_pContainer->getPosition().y + moveDistance.y;
					
					m_tScrollDistance = moveDistance;
					this->setContentOffset(CCPoint(newX, newY));
				}
			}
		}
		m_bIsScrolling = true;
	}

当前cocos版本：2.2.3，参考cocos版本：3.4.x
