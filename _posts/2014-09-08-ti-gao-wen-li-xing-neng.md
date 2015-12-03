---
layout: post
title: "cocos2d-x中提高性能的缓存解析"
date:   2014-09-08
categories: Lua
---

* content
{:toc}

游戏最常见的就是动态加载纹理。但是动态加载纹理就意味着内存不断的分配销毁内存，纹理过多的话意味着内存占用就更高。一旦纹理比较多且加载不及时还会遇到某些纹理已经加载而某些纹理还没有加载（会在某个矩形区域内呈黑色）。纹理的加载与销毁都是需要时间与空间的，这对于游戏是一个挑战，因为游戏世界里是不断的在变化的。当然啦，这就需要程序员对程序的优化啦:)

cocos2d-x中早已提供了这些东西方面我们做优化，那就是缓存，能够在一定程度上提高纹理或者动画的加载性能，避免内存浪费。  

####```TextureCache```###
TextureCache纹理缓存是最底层也是最有效的纹理缓存。它到底有什么用呢？假设游戏中有个界面用到的图片非常多，第一次点进这界面时速度非常慢（因为要加载绘制很多图片），可第二次点击却一下子就进去了。这是为什么呢？原来Cocos2dx的渲染机制是可以重复使用同一份纹理在不同的场合进行绘制，从而达到重复使用，降低内存和GPU运算资源的消耗与开销。比如：  

    auto sprite = Sprite::create("box.png");

在```Sprite::create("box.png")```调用了```Sprite::initWithFile("box.png")```,看下```initWithFile```的代码：  

    bool Sprite::initWithFile(const std::string& filename)
    {
        CCASSERT(filename.size()>0, "Invalid filename for sprite");
    	// 将box.png添加到纹理缓存中
        Texture2D *texture = Director::getInstance()->getTextureCache()->addImage(filename);
        if (texture)
        {
            Rect rect = Rect::ZERO;
            rect.size = texture->getContentSize();
            return initWithTexture(texture, rect);
        }
    
        // don't release here.
        // when load texture failed, it's better to get a "transparent" sprite then a crashed program
        // this->release();
        return false;
    }

    // 在TextureCache::addImage中实际上先从一个缓存列表中找是否存在box.png，如果存在就直接返回已经存在的纹理，如果没有，那就创建一个纹理
    Texture2D * TextureCache::addImage(const std::string &path)
    {
        Texture2D * texture = nullptr;
        Image* image = nullptr;
    	
    	// 获得box.png的全路径
        std::string fullpath = FileUtils::getInstance()->fullPathForFilename(path);
        if (fullpath.size() == 0)
        {
            return nullptr;
        }
    	// 然后再缓存map中查找box.png这个纹理
        auto it = _textures.find(fullpath);
        if( it != _textures.end() )
            texture = it->second;
    	
    	// 如果没有找到box.png这个纹理
        if (! texture)
        {
            do 
            {
                image = new Image();
                CC_BREAK_IF(nullptr == image);
    			
                bool bRet = image->initWithImageFile(fullpath);
                CC_BREAK_IF(!bRet);
    			
    			// 新建一个纹理
                texture = new Texture2D();
    
                if( texture && texture->initWithImage(image) )
                {
    				// 并将这个纹理添加到纹理缓存中，便于第二次使用
                    _textures.insert( std::make_pair(fullpath, texture) );
                }
                else
                {
                    CCLOG("cocos2d: Couldn't create texture for file:%s in TextureCache", path.c_str());
                }
            } while (0);
        }
    
        CC_SAFE_RELEASE(image);
    
        return texture;
    }

综合上面来看，当第一次create一个box.png纹理的时候，会新建纹理，在第二次create box.png纹理，那么会直接从缓存列表中取出来，相比第一次，第二次的速度非常快，空间占用也少。在代码中我们可以这样优化：  

    void Scene::init()
    {
    	// 预先将纹理加载到游戏中来，就是所谓的 资源预加载 啦
    	Director::getInstance()->getTextureCache()->addImage("box.png", "box.png");
    	Director::getInstance()->getTextureCache()->addImage("man.png", "man.png");
    	Director::getInstance()->getTextureCache()->addImage("jump.png", "jump.png");
    }
    
    // 然后再需要使用的地方
    void Scene::showImage()
    {
    	auto boxTexture = Director::getInstance()->getTextureCache()->getTextureForKey("box.png");
    	// 这样即使在第一次使用该纹理时速度就会非常快
    	auto box = Sprite::createWithTexture(boxTexture);
    }

这里有一个会影响游戏体验的建议就是资源加载，进度条加载资源。在游戏开始或者关卡开始前预先加载所有的资源。  

####```SpriteFrameCache```####
精灵帧缓存```SpriteFrameCache```就是对```TextureCache```的封装，只不过它缓存的是精灵帧，即纹理指定区域的矩形块。而```TextureCache```缓存的是原始纹理图。```SpriteFrameCache```最常用的就是解析一张由N多小图拼成的大图，它会通过与大图一并生成的plist文件进行小图解析，缓存下指定区域的矩形块。有一张大图如下：  

![alt text](/img/2014-09-09.png)  

拼接这张大图生成的plist文件如下：  

    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
        <dict>
            <key>frames</key>
            <dict>
                <key>stickman1.png</key>
                <dict>
                    <key>frame</key>
                    <string>\{\{0,0\},\{60,80\}\}</string>
                    <key>offset</key>
                    <string>\{0,0\}</string>
                    <key>rotated</key>
                    <false/>
                    <key>sourceColorRect</key>
                    <string>\{\{0,0\},\{60,80\}\}</string>
                    <key>sourceSize</key>
                    <string>\{60,80\}</string>
                </dict>
                <key>stickman2.png</key>
                <dict>
                    <key>frame</key>
                    <string>\{\{0,80\},\{60,80\}\}</string>
                    <key>offset</key>
                    <string>\{0,0\}</string>
                    <key>rotated</key>
                    <false/>
                    <key>sourceColorRect</key>
                    <string>\{\{0,0\},\{60,80\}\}</string>
                    <key>sourceSize</key>
                    <string>\{60,80\}</string>
                </dict>
                <key>stickman3.png</key>
                <dict>
                    <key>frame</key>
                    <string>\{\{0,160\},\{60,80\}\}</string>
                    <key>offset</key>
                    <string>\{0,0\}</string>
                    <key>rotated</key>
                    <false/>
                    <key>sourceColorRect</key>
                    <string>\{\{0,0\},\{60,80\}\}</string>
                    <key>sourceSize</key>
                    <string>\{60,80\}</string>
                </dict>
            </dict>
            <key>metadata</key>
            <dict>
                <key>format</key>
                <integer>2</integer>
                <key>realTextureFileName</key>
                <string>man.png</string>
                <key>size</key>
                <string>\{60,240\}</string>
                <key>smartupdate</key>
                <string>$TexturePacker:SmartUpdate:12e7183ac6818f625b15525e60ac4b63$</string>
                <key>textureFileName</key>
                <string>man.png</string>
            </dict>
        </dict>
    </plist>

精灵帧缓存```SpriteFrameCache```会解析这个plist文件，进而知道大图里面每一张小图的位置和大小，这样就可以缓存相应的矩形。伪代码：  

    // 使用SpriteFrameCache缓存
    SpriteFrameCache::getInstance()->addSpriteFramesWithFile("man.plist");
    
    // addSpriteFramesWithFile的源代码
    void SpriteFrameCache::addSpriteFramesWithFile(const std::string& plist)
    {
        CCASSERT(plist.size()>0, "plist filename should not be nullptr");
    
        if (_loadedFileNames->find(plist) == _loadedFileNames->end())
        {
            std::string fullPath = FileUtils::getInstance()->fullPathForFilename(plist);
            ValueMap dict = FileUtils::getInstance()->getValueMapFromFile(fullPath);
    		
    		// 获得大图的全路径
            string texturePath("");
    
            if (dict.find("metadata") != dict.end())
            {
                ValueMap& metadataDict = dict["metadata"].asValueMap();
                // try to read  texture file name from meta data
                texturePath = metadataDict["textureFileName"].asString();
            }
    
            if (!texturePath.empty())
            {
                // build texture path relative to plist file
                texturePath = FileUtils::getInstance()->fullPathFromRelativeFile(texturePath.c_str(), plist);
            }
            else
            {
    			// 如果plist文件存在，就找到了plist文件对应的大图
                // build texture path by replacing file extension
                texturePath = plist;
    
                // remove .xxx
                size_t startPos = texturePath.find_last_of("."); 
                texturePath = texturePath.erase(startPos);
    
                // append .png
    			// 得到man.plist文件对应的man.png图片
                texturePath = texturePath.append(".png");
    
                CCLOG("cocos2d: SpriteFrameCache: Trying to use file %s as texture", texturePath.c_str());
            }
    		
    		// 加载大图纹理
            Texture2D *texture = Director::getInstance()->getTextureCache()->addImage(texturePath.c_str());
    
            if (texture)
            {
    			// 将大图纹理添加到精灵帧缓存中
                addSpriteFramesWithDictionary(dict, texture);
    			// 添加plist文件到缓存列表中
                _loadedFileNames->insert(plist);
            }
            else
            {
                CCLOG("cocos2d: SpriteFrameCache: Couldn't load texture");
            }
        }
    }
    
    /*
     addSpriteFramesWithDictionary代码比较长，这里就不贴出来了，addSpriteFramesWithDictionary中主要是根据解析大图纹理texture，分别将大图里面的小图创建SpriteFrame存放在_spriteFrames这个map里面，key值就是小图对应的文件名。
    */
    
    // 在使用的地方
    void Scene::showImage()
    {
    	auto sprite = Sprite::createWithSpriteFrameName("box.png");
    }
    
    // createWithSpriteFrameName的源代码
    Sprite* Sprite::createWithSpriteFrameName(const std::string& spriteFrameName)
    {
        SpriteFrame *frame = SpriteFrameCache::getInstance()->getSpriteFrameByName(spriteFrameName);
        // ...
        return createWithSpriteFrame(frame);
    }
    
    // 跳转到getSpriteFrameByName的源代码
    SpriteFrame* SpriteFrameCache::getSpriteFrameByName(const std::string& name)
    {
    	/*
    		在_spriteFrames中查找纹理，在addSpriteFramesWithDictionary函数中已经将大图纹理分解成多个小图纹理存放在_spriteFrames中。（这里的说法欠妥，_spriteFrames中存放的实际上是精灵帧，是一个纹理大小的矩形）
    	*/
        SpriteFrame* frame = _spriteFrames.at(name);
        if (!frame)
        {
            // try alias dictionary
            std::string key = _spriteFramesAliases[name].asString();
            if (!key.empty())
            {
                frame = _spriteFrames.at(key);
                if (!frame)
                {
                    CCLOG("cocos2d: SpriteFrameCache: Frame '%s' not found", name.c_str());
                }
            }
        }
        return frame;
    }

当需要加载一张纹理的时候，如果```TextureCache```中没有时，会去创建一张。但是```SpriteFrameCache```中没有相关纹理时不会去加载纹理，原因是因为找不到大图，何来的小图？所以当使用```Sprite::createWithSpriteFrameName```创建一个Sprite的时候，确保plist以及相应的大图已成功加载到缓存中。  

####```AnimationCache```####
动画缓存。对于精灵动画，每次创建时都需要加载精灵帧，然后按顺序添加到数组，再用Animation读取数组创建动画。这是一个非常烦琐的计算过程。而对于使用频率高的动画，例如角色的走动，可以将其加入到```AnimationCache```中，每次使用都从这个缓存中调用，这样可以有效的降低创建动画的巨大消耗。

其原理和上面两种缓存类似，预先创建放置在map中，在使用时直接去map中找到即可，这里就不贴源代码了。  

    // 加入缓存列表
    AnimationCache::getInstance()->addAnimation(repeatRun,"run");
    
    // 从缓存中获取动画
    auto repeatRun = AnimationCache::getInstance()->animationByName("run");

好了，就酱。