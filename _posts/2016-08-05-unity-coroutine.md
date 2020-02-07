---
layout: post
title: "减少Coroutine带来的GC开销"
date:   2016-08-05
categories: Game-Dev
---

游戏里面对于异步的处理经常会用到Coroutine，很多时候Coroutine并不是一次处理或一帧内处理完毕，所以经常会用到```yield return```：  

    yield return new WaitForEndOfFrame()
    yield return new WaitForFixedUpdate();
    yield return new WaitForSeconds(1.0f);

众所周知，C#是带有GC的语言，在yield return中出现了```new```也就意味着可能会带来一次GC，那么每一次的yield return都会产生新对象。在Unity中，我们可以保存对象的方式来避免对象的分配。

    public static class Yielders
    {
        class FloatComparer : IEqualityComparer<float>
        {
            bool IEqualityComparer<float>.Equals(float x, float y)
            {
                return x == y;
            }
            int IEqualityComparer<float>.GetHashCode(float obj)
            {
                return obj.GetHashCode();
            }
        }
    
        static WaitForEndOfFrame _endOfFrame = new WaitForEndOfFrame();
        public static WaitForEndOfFrame EndOfFrame
        {
            get { _endOfFrame; }
        }
    
        static WaitForFixedUpdate _fixedUpdate = new WaitForFixedUpdate();
        public static WaitForFixedUpdate FixedUpdate
        {
            get { _fixedUpdate; }
        }
    
        static Dictionary<float, WaitForSeconds> _timeInterval = new Dictionary<float, WaitForSeconds>(100, new FloatComparer());
        public static WaitForSeconds GetWaitForSeconds(float seconds)
        {
            WaitForSeconds wfs;
            if (!_timeInterval.TryGetValue(seconds, out wfs))
                _timeInterval.Add(seconds, wfs = new WaitForSeconds(seconds));
    
            return wfs;
        }
    }
