---
title: nonatomic 和 atomic
date: 2020-11-20 16:14:17
tags: 线程
---
# nonatomic 和 atomic

atomic 并不是绝对线程安全，它能保证代码进入 getter 和 setter 方法的时候是安全的，但是并不能保证多线程的访问情况下是安全的，一旦出了 getter 和 setter 方法，其线程安全就要由程序员自己来把握，所以 atomic 属性和线程安全并没有必然联系。


## nonatomic 特点
* 不会加锁
* 多线程给nonatomic属性赋值，是可能重复release而崩溃的


```
@interface NonatomicTest : NSObject
@property (nonatomic, strong) id obj;
@end
@implementation
+ (void)main {
    // 多线程给 nonatomic 属性赋值，是可能重复 release 而崩溃的
    // ivar 所指向的内存区域被其他线程修改了，产生了野指针. Thread 62: EXC_BAD_ACCESS (code=1, address=0x43308abb42f0)
    for (int i = 0; i < 100000; ++i) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            id o = [[NSObject alloc] init];
            self.obj = o;
        });
    }  
}
@end
```

## atomic 特点
* 无差别加锁，及时是在非多线程环境下，加锁会损耗性能。
* 由于是原子操作，可以在多线程情况下不崩溃，但是无法保证业务正确。

## 为什么 atomic 可以在多线程环境下可以不崩溃而 nonatomic 会崩溃？
atomic 的底层实现是有加锁的，老版本是自旋锁，iOS10 开始是互斥锁，spinlock 底层实现改变了。
nonatomic 底层是没有加锁的, 多线程读写，资源抢夺就会崩，比如多线程给属性赋值，是可能重复 release 而崩溃的。


```
// objc4-779.1 
// @property (strong) strong 属性赋值底层实现
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {// 偏移 0 就是修改 isa 指针
        object_setClass(self, newValue);
        return;
    }
    
    id oldValue;
    id *slot = (id*) ((char*)self + offset);// 根据偏移找到变量地址
    
    if (*slot == newValue) return;// 1、值不相等，才做赋值操作
    newValue = objc_retain(newValue);// 2、先对新值 retain
    
    if (!atomic) {// nonatomic
        oldValue = *slot;// 3、保存变量旧值
        *slot = newValue;// 4、给变量赋值新值
    } else {// atomic
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();// 加锁 由于这里有加锁逻辑，那么多线程情况下是不会多次获取到同一个旧值的，所以也不会引起重复释放的问题
        oldValue = *slot;// 3、保存变量旧值
        *slot = newValue;// 4、给变量赋值新值        
        slotlock.unlock();// 解锁
    }
    
    objc_release(oldValue);// 5、release 旧值
}
```

