---
layout: post
title: "FaceUI<1>: Direct2D渲染异形窗口"
date:   2015-11-20 00:00:00
categories: C++
excerpt: FaceUI<1>: Direct2D渲染异形窗口
---

####前言
最近计划参照DuiLib重写为FaceUI，以解决DuiLib中诸多不爽的问题。FaceUI中将利用C++11实现更加完善方便的UI事件机制，同时采用TinyXML作为XML解析器，将重要工作交付给更加快速稳定的第三方库。至于渲染引擎方面，DuiLib为了兼容XP采用了GDI（GDI+的效率问题使得不在本文的讨论范围之内）使得没办法实现更漂亮的UI，由于GDI不含alpha通道无法原生实现异形窗口，即使修改DuiLib实现异形会造成其他绘制操作不含alpha操作导致绘图出现“穿透”。

为了解决渲染引擎上的天生不足，所以FaceUI中选择Direct2D（D2D）作为核心渲染引擎，在FaceUI的预研阶段，首先便用
D2D实现异形窗口。

####异形窗口的D2D实现
其实GDI本身是可以实现异形窗口的，无非就是创建一个兼容位图然后进行相应的绘制，最后UpdateLayeredWindow使窗口异形。这种方式带来的缺点很明显，当在异形窗口上进行诸如LineTo等绘图操作，由于画笔中缺失alpha通道，导致绘图出现穿透现象。解决办法也很简单但也繁琐：创建画笔时加上alpha通道值，虽然简单但是繁琐！

但D2D可以从根本上解决这个问题。

在D2D中，各种绘图操作诸如DrawBitmap、DrawLine等操作是绘制在一个RenderTarget上的，再由RenderTarget显示在相关的设备上。D2D中有两种设备Target：HWNDRenderTarget和DCRenderTarget，很明显前者绑定一个窗口的HWND进行绘制，后者则可以绑定一个DC进行绘制，不限制于窗口的DC。

在研究过程中，HWNDRenderTarget可以通过其内部的GdiInteropRenderTarget实现异形窗口，但异形之后无法再进行任何绘制操作了，关于这个问题至今未找到明确的资料。

采用DCRenderTarget实现异形比较简单：绑定窗口的DC，然后利用窗口DC创建一个兼容DC和一张兼容位图，然后将兼容位图选入兼容DC内，然后Target绑定这个兼容DC，接下来就是进行D2D的绘制了，绘制完成后调用UpdateLayeredWindow即可实现。虽然简单，但是我却在摸索D2D实现异形窗口过程中花了足足两周才完美实现。

异形窗口一般都是以一张含有alpha通道的png图片作为背景，既然采用了D2D作为渲染，自然解析图片就采用WIC了。

``` c++

HRESULT LoadImageFile(IWICImagingFactory *pIWICFactory,
						PCWSTR uri,
						UINT destinationWidth,
						UINT destinationHeight)
{
	HRESULT hRet = S_OK;

	IWICBitmapDecoder		*pDecoder = nullptr;
	IWICBitmapFrameDecode	*pSource = nullptr;
	IWICStream				*pStream = nullptr;
	IWICFormatConverter		*pConverter = nullptr;
	IWICBitmapScaler		*pScaler = nullptr;

	hRet = pIWICFactory->CreateDecoderFromFilename(uri, nullptr, GENERIC_READ, WICDecodeMetadataCacheOnLoad, &pDecoder);
	if (SUCCEEDED(hRet))
	{
		hRet = pDecoder->GetFrame(0, &pSource);
	}

	if (SUCCEEDED(hRet))
	{
		hRet = pIWICFactory->CreateFormatConverter(&pConverter);
	}


	UINT originalWidth, originalHeight;
	hRet = pSource->GetSize(&originalWidth, &originalHeight);
	if (SUCCEEDED(hRet))
	{
		if (destinationWidth != 0 && destinationHeight != 0)
		{
			originalWidth = destinationWidth;
			originalHeight = destinationHeight;
		}

		hRet = pIWICFactory->CreateBitmapScaler(&pScaler);
		if (SUCCEEDED(hRet))
		{
			hRet = pScaler->Initialize(pSource, originalWidth, originalHeight, WICBitmapInterpolationModeCubic);
		}

		if (SUCCEEDED(hRet))
		{
			hRet = pConverter->Initialize(pScaler, GUID_WICPixelFormat32bppPBGRA,
				WICBitmapDitherTypeNone,
				nullptr,
				0.f,
				WICBitmapPaletteTypeMedianCut);
		}
	}
	if (SUCCEEDED(hRet))
	{
		hRet = dcRenderTarget->CreateBitmapFromWicBitmap(pConverter, nullptr, &gBitmap);
	}

	if (SUCCEEDED(hRet))
	{
		hRet = pIWICFactory->CreateBitmapFromSource(pConverter, WICBitmapCacheOnLoad, &gWicBitmap);
	}

	SAFE_RELEASE(pDecoder);
	SAFE_RELEASE(pSource);
	SAFE_RELEASE(pStream);
	SAFE_RELEASE(pConverter);
	SAFE_RELEASE(pScaler);

	return hRet;
}

```

当窗口响应WM_PAINT时：

``` c++

void Render2(HWND hwnd)
{
	RECT rc;
	ZeroMemory(&rc, sizeof(rc));
	RECT rcClient;
	::GetWindowRect(hwnd, &rcClient);
	SIZE wndSize = { rcClient.right - rcClient.left, rcClient.bottom - rcClient.top };
	HDC hwndDC = GetDC(hwnd);

	gDC = ::CreateCompatibleDC(hwndDC);
	HBITMAP memBitmap = ::CreateCompatibleBitmap(hwndDC, wndSize.cx, wndSize.cy);
	::SelectObject(gDC, memBitmap);
	dcRenderTarget->BindDC(gDC, &rcClient);
	dcRenderTarget->BeginDraw();
	dcRenderTarget->DrawLine(D2D1::Point2F(200, 200), D2D1::Point2F(300, 300), gBrush, 50);
	dcRenderTarget->DrawLine(D2D1::Point2F(100, 100), D2D1::Point2F(100, 300), gBrush, 50);
	dcRenderTarget->DrawLine(D2D1::Point2F(0, 0), D2D1::Point2F(100, 600), gBrush);
	dcRenderTarget->DrawBitmap(gBitmap, D2D1::RectF(0, 0, 68, 68), 1.0f);
	dcRenderTarget->EndDraw();
	POINT ptDest = { rcClient.left, rcClient.top };
	POINT ptSrc = { 0, 0 };
	SIZE szLayered = { rcClient.right - rcClient.left, rcClient.bottom - rcClient.top };
	BLENDFUNCTION bf;
	bf.AlphaFormat = AC_SRC_ALPHA;
	bf.BlendFlags = 0;
	bf.BlendOp = AC_SRC_OVER;
	bf.SourceConstantAlpha = 255;
	::UpdateLayeredWindow(hwnd, hwndDC, &ptDest, &szLayered, gDC, &ptSrc, RGB(0, 0, 0), &bf, ULW_ALPHA);
	::ReleaseDC(nullptr, gDC);
	::ReleaseDC(hwnd, hwndDC);
}

```

至此变采用D2D完美实现了异形窗口，且暂无任何其他不良反应。最终实现代码仅有这么多，但是找到这种实现方式的过程却异常曲折，再此就不表了。

效果图一张：  
![alt text](/img/2015-11-21.png)  