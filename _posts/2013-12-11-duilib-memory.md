---
layout: post
title: "duilib里的内存回收"
categories: c++
---

我们的刺猬助手客户端的界面是用Duilib写的，前天在调试代码的时候发现一个问题，在关闭应用程序的时候发现程序崩溃了，原因一想便得知，肯定是哪里的指针重复释放了。在跟踪到主窗口的析构函数中，中断到了这样的一句代码：  

``` c++
if ( pControl != nullptr )
{
     delete pControl;
     pControl = nullptr;
}
```

显然就是这个pControl被重复释放了，而这个pControl是我在程序里通过FindControl得到的，既然这个被重复释放了，那么可以肯定的是这个pControl在某处被释放掉了。就这样粗心添加的一句代码就导致了关闭程序崩溃的问题，为此决定深入分析一下duilib里面的内存回收。

duilib中对于界面元素可以分为 容器-Container，布局-HorizontalLayout/VerticalLayout，控件-Control。在duilib中，容器是最基础的，也就是说容器中能添加各种东西，能够在容器中添加布局和控件。然后我们想到另外一个方面duilib里界面上所有的元素都是通过xml解析出来的，在UIDlgBuilder.h/cpp中有一个工厂方法，就是解析xml的标签元素，然后通过new的方式将控件/布局/容器创建出来，然后根据xml中的嵌套关系add到相对应的容器中，最后得到的是一条最顶端为window的控件链路。
看Container里面对Add的实现：  

``` c++
bool CContainerUI::Add(CControlUI* pControl)
{
     if( pControl == NULL) return false;

     if( m_pManager != NULL ) m_pManager->InitControls(pControl, this);
     if( IsVisible() ) NeedUpdate();
     else pControl->SetInternVisible(false);
     return m_items.Add(pControl);  
}

bool CContainerUI::AddAt(CControlUI* pControl, int iIndex)
{
     if( pControl == NULL) return false;

     if( m_pManager != NULL ) m_pManager->InitControls(pControl, this);
     if( IsVisible() ) NeedUpdate();
     else pControl->SetInternVisible(false);
     return m_items.InsertAt(iIndex, pControl);
}
```

从上面可以看到，在InitControls完了之后将在Manager中new出来的控件指针添加到了一个叫m_items的变量中，m_items变量的类型是CstdPtrArray，这是duilib的扩展类型，就是放置控件指针的List，从上面可以看到，在manager里new出来的东西最终放到了CStdPtrArray中，猜想是用作析构。  

再来看主窗体的析构：CWindowWnd成员函数，__WndProc/__ControlProc负责最后的析构，在这两个函数里将最后的处理工作分发到OnFinalMessage里，我在主窗口里实现这个虚函数：  

``` c++
void CPurpleShellMainWnd::OnFinalMessage( HWND hWnd )
{
     try
     {
          WindowImplBase::OnFinalMessage(hWnd);
          delete this;
     }
     catch (...)
     {
          throw "CPurpleShellMainWnd::OnFinalMessage";
     }
}
```

在delete this的引发析构，进而引发manager的析构，于是我们回到manager的析构中：  

``` c++
CPaintManagerUI::~CPaintManagerUI()
{
    // Delete the control-tree structures
    for( int i = 0; i < m_aDelayedCleanup.GetSize(); i++ ) delete static_cast<CControlUI*>(m_aDelayedCleanup[i]);
    for( int i = 0; i < m_aAsyncNotify.GetSize(); i++ ) delete static_cast<TNotifyUI*>(m_aAsyncNotify[i]);
    m_mNameHash.Resize(0);
     delete m_pRoot;
     /*if ( m_pRoot != NULL )
     {
          delete m_pRoot;
          m_pRoot = NULL;
     }*/

    ::DeleteObject(m_DefaultFontInfo.hFont);
    RemoveAllFonts();
    RemoveAllImages();
    RemoveAllDefaultAttributeList();
    RemoveAllOptionGroups();
    RemoveAllTimers();

    // Reset other parts...
    if( m_hwndTooltip != NULL ) ::DestroyWindow(m_hwndTooltip);
    if( m_hDcOffscreen != NULL ) ::DeleteDC(m_hDcOffscreen);
    if( m_hDcBackground != NULL ) ::DeleteDC(m_hDcBackground);
    if( m_hbmpOffscreen != NULL ) ::DeleteObject(m_hbmpOffscreen);
    if( m_hbmpBackground != NULL ) ::DeleteObject(m_hbmpBackground);
    if( m_hDcPaint != NULL ) ::ReleaseDC(m_hWndPaint, m_hDcPaint);
    m_aPreMessages.Remove(m_aPreMessages.Find(this));
}
```

对于new出来的东西这句就是析构 delete static_cast<CControlUI*>(m_aDelayedCleanup[i]);，但是m_aDelayedCleanup的内容怎么来的呢？，析构会引发控件的析构，那么去控件的析构看：  

``` c++
CContainerUI::~CContainerUI()
{
     m_bDelayedDestroy = false;
     RemoveAll();
     if( m_pVerticalScrollBar ) delete m_pVerticalScrollBar;
     if( m_pHorizontalScrollBar ) delete m_pHorizontalScrollBar;
}
```

原来是RemoveAll做了工作，再转过去看RemoveAll的内容：  

``` c++
void CContainerUI::RemoveAll()
{
     for( int it = 0; m_bAutoDestroy && it < m_items.GetSize(); it++ ) {
          if( m_bDelayedDestroy && m_pManager ) m_pManager->AddDelayedCleanup(static_cast<CControlUI*>(m_items[it]));            
          else delete static_cast<CControlUI*>(m_items[it]);
     }
     m_items.Empty();
     NeedUpdate();
}
```

由此可见，在Remove的时候，容器将自身的控件链传递给manager中的AddDelayedCleanup，在AddDelayedCleanup里又将控件指针添加到m_aDelayedCleanup，由于给个容器都由自己的一条控件链，这样子递归析构，保证根据xml中new出来的容器能够顺利被析构。

对于字体的回收，默认属性链表，图像的回收都在manager的析构函数里，有兴趣的人可以自己去看看。

1.根据xml解析new出来的对象，然后你去FindControl它的时候，不要手动去delete它。  
2.动态在程序中new出来的，如果你没有add到某个容器里，最好自己去delete它，一旦你将它add到某个容器里，那么它就存在那个容器的控件链中，duilib会去将会帮助你回收。