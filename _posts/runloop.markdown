---
layout: post
title: runloop源码学习
date: 2018
---

### runloop创建函数
在调用CFRunLoopGetMain和CFRunLoopGetCurrent函数时，如果线程没有创建runloop，会去调用下面的函数去创建runloop

``` C
// should only be called by Foundation
// t==0 is a synonym for "main thread" that always works
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    // kNilPthreadT为0，如果t为0，就赋值为主线程
    if (pthread_equal(t, kNilPthreadT)) {
	    t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    // __CFRunLoops为一个全局字典，key为线程，value为线程对应的runloop
    // 这段为初始化__CFRunLoops，初始化时创建主线程对应的runloop，并存储到__CFRunLoops中，
    // 除了在__CFRunLoops中记录了线程与runloop的关系外，在创建runloop时，会设置loop->_pthread = t;也标记了此runloop的thread
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
        CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
        if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
            CFRelease(dict);
        }
        CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);
    // 创建线程t的runloop，并保存到__CFRunLoops
    if (!loop) {
	    CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
        loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
        if (!loop) {
            CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
            loop = newLoop;
        }
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
	    CFRelease(newLoop);
    }
    // _CFSetTSD中会调用static __CFTSDTable *__CFTSDGetTable()函数，此函数会获取或创建线程本地空间，返回一个名为table的__CFTSDTable结构，里面有三项destructorCount，data[CF_TSD_MAX_SLOTS]以及destructors[CF_TSD_MAX_SLOTS]
    // void *_CFSetTSD(uint32_t slot, void *newVal, tsdDestructor destructor)会把loop也就是newVal存到table->data[slot]中，把析构函数也就是destructor存到table->destructors[slot]中。并返回table->data[slot]中的旧值
    // void *_CFGetTSD(uint32_t slot) 返回table->data[slot]
    // 在YY的博客看到此函数的说明是：注册一个回调，当线程销毁时，顺便也销毁其对应的 RunLoop。
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```