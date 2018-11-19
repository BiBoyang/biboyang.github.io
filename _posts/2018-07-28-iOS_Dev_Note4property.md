---
layout: post
title: property的研究（二）：weak关键字
date: 2018-07-28 
tags: iOS
---

## weak的实现

我们这里直接查看[	objc4-723.tar.gz](https://opensource.apple.com/tarballs/objc4/)源码。
节省点话说，可以分为以下三步：

1、初始化时：runtime会调用**objc_initWeak**函数，初始化一个新的weak指针指向对象的地址。

2、添加引用时：**objc_initWeak**函数会调用 **objc_storeWeak()** 函数， **objc_storeWeak()** 的作用是更新指针指向，创建对应的弱引用表。

3、释放时，调用**clearDeallocating**函数。**clearDeallocating**函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。


## 实现过程
当有一个weak的属性时。编译器会自动创建一下方法
```
objc_initWeak(&obj1,obj);//初始化
objc_destroyWeak(&obj1);//释放
```
在**NSObject.mm**文件中，找到方法的实现
```
id objc_initWeak(id *location, id newObj)
{
    // 查看对象实例是否有效
    // 无效对象直接导致指针释放
    if (!newObj)
    {
        *location = nil;
        return nil;
    }
    
    // 这里传递了三个 bool 数值
    // 使用 template 进行常量参数传递是为了优化性能
    // DontHaveOld--没有旧对象，
    // DoHaveNew--有新对象，
    // DoCrashIfDeallocating-- 如果newObj已经被释放了就Crash提示
    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
```
这里方法比较简单明了，但是我们要知道这里有一个潜在的前提条件：
> location要是一个没有被注册为__weak对象的有效指针。如果newObj是空指针或它指向的对象已经释放，则location也就是weak的指针将初始化为0（nil）。 否则，将object注册为指向location的__weak对象。 
这里是表层的判断，我们继续往下看相关实现

```
// 更新weak变量.
// 当设置HaveOld是true，即DoHaveOld，表示这个weak变量已经有值，需要被清理，这个值也有能是nil
// 当设置HaveNew是true， 即DoHaveNew，表示有一个新值被赋值给weak变量，这个值也有能是nil
//当设置参数CrashIfDeallocating是true，即DoCrashIfDeallocating，如果newObj已经被释放或者newObj是一个不支持弱引用的类，则暂停进程
// deallocating或newObj的类不支持弱引用
// 当设置参数CrashIfDeallocating是false，即DontCrashIfDeallocating，则存储nil

enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id 
storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    
    // 初始化当前正在 +initialize 的类对象为nil
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    
    // 声明新旧SideTable，
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:   
    // 如果weak ptr之前弱引用过一个obj，则将这个obj所对应的SideTable取出，赋值给oldTable
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.
    //通过确保没有弱引用的对象具有未初始化的 isa，防止弱引用机制和 +initialize 机制之间的死锁。

    if (haveNew  &&  newObj) {
        // 获得新对象的 isa 指针
        Class cls = newObj->getIsa();
        // 判断 isa 非空且已经初始化
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            // 解锁新旧SideTable
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            // 如果 newObj 已经完成执行完 +initialize 是最理想情况
            // 如果 newObj的 +initialize 仍然在线程中执行
            // (也就是说newObj的 +initialize 正在调用 storeWeak 方法)
            // 通过设置previousInitializedClass以在重试时识别它。
            

            previouslyInitializedClass = cls;

            goto retry;
        }
    }

    // Clean up old value, if any.
    // 清除旧值，实际上是清除旧对象weak_table中的location

    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any.
    // 分配新值，实际上是保存location到新对象的weak_table种

    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.
        // 如果弱引用被释放 weak_register_no_lock 方法返回 nil
        
        // 如果新对象存在，并且没有使用TaggedPointer技术，在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            // 标记新对象有weak引用，isa.weakly_referenced = true;
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        // 设置location指针指向newObj
        // 不要在其他地方设置 *location。 那会引起竞争
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj;
}
```
这个函数的作用是在添加引用的时候，添加新的指针和创建对应的弱引用表。
这里有几个关键方法，需要说明一下。
#### SideTable
这个是一个结构体。
```
enum HaveOld { DontHaveOld = false, DoHaveOld = true };
enum HaveNew { DontHaveNew = false, DoHaveNew = true };

struct SideTable {
    //原子操作自旋锁
    spinlock_t slock;
    // 引用计数的 hash 表
    RefcountMap refcnts;
    // weak 引用全局 hash 表
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<HaveOld, HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<HaveOld, HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
```
这里面第一个是为了防止竞争选择的自旋锁，第二个是协助对象的 isa 指针的 extra_rc 共同引用计数的变量，第三个就是我们要了解的关键。
```
struct weak_table_t {
    // 保存了所有指向指定对象的 weak 指针
    weak_entry_t *weak_entries;
    // 存储空间
    size_t    num_entries;
    // 参与判断引用计数辅助量
    uintptr_t mask;
    // hash key 最大偏移值
    uintptr_t max_hash_displacement;
};
```
我们继续往下看
```
/**
 * The internal structure stored in the weak references table.
 //存储在弱引用表中的内部结构
 * It maintains and stores
 用来维护和存储
 * a hash set of weak references pointing to an object.
 指向对象的弱引用的哈希集
 * If out_of_line_ness != REFERRERS_OUT_OF_LINE then the set
 * is instead a small inline array.
  如果out_of_line_ness 不等于REFERRERS_OUT_OF_LINE，然后这个集合会被一个小的内联数组替代。
 */
#define WEAK_INLINE_COUNT 4

// out_of_line_ness field overlaps with the low two bits of inline_referrers[1].
//out_of_line_ness 的字段与低两位的inline_referrers[1]部分重叠
// inline_referrers[1] is a DisguisedPtr of a pointer-aligned address.
// inline_referrers[1]是一个指针对齐地址的DisguisedPtr
// The low two bits of a pointer-aligned DisguisedPtr will always be 0b00
// (disguised nil or 0x80..00) or 0b11 (any other address).
// 一个指针对齐地址的DisguisedPtr的低两位将地址将会变成 0b00（伪装的nil）或者0b11.
// Therefore out_of_line_ness == 0b10 is used to mark the out-of-line state.
// 因此out_of_line_ness == 0b10 被用于标记离线状态。
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line_ness : 2;
            uintptr_t        num_refs : PTR_MINUS_2;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent)
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
```
 弱表是由单个自旋锁控制的哈希表。一个被分配的内存块，大多数是一个对象，但是在GC之下，任何这样的分配，可以是它的地址存储一个弱引用存储单元中，通过使用编译器生成的写屏障或手工编码的寄存器弱原语的使用。
 与注册相关联是一个回调块，应对这种情况：其中一个被分配的内存块被回收。该表在分配内存的地址上被哈希。当弱引用标记内存改变它的引用，我们可以查看之前的引用。
 因此，在哈希表中，由弱引用项索引的是当前存储该地址的所有位置的列表。
 对于ARC，我们还跟踪是否存在一个任意被解除分配的对象，在调用dealloc之前将其简单地放置在表中，以及在内存回收之前释放objc_clear_deallocating。
 
 我们在上边的代码中可以发现有两个**weak_referrer_t**，第一个应该是我们正常情况下的weak表，第二个我有点没看明白，但是根据上下文，猜测可能是一个补充，在当前弱引用对象少于2个的时候，不在采用hash了，直接用数组去实现的。
 这里确实有点难懂。
 里面还有些细节比如bad_weak_table神马的，有时间继续往下研究。要好好补一补数据结构的知识了。
 这里直接那冬瓜的图片来总结SideTable。
 ![](http://7xwh85.com1.z0.glb.clouddn.com/sidetable.png)
 里面还有旧对象解除注册操作 weak_unregister_no_lock和新对象添加注册操作 weak_register_no_lock。
 ```
 id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
------------
id weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        append_referrer(entry, referrer);
    } 
    else {
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
 ```
 创建流程如下。
 ![](http://7xwh85.com1.z0.glb.clouddn.com/weaktable-2.png)
 
 
## 销毁过程
释放对象的时候，基本流程如下
>1、调用objc_release
2、因为对象的引用计数为0，所以执行dealloc
3、_objc_rootDealloc
4、object_dispose
5、objc_destructInstance
6、objc_clear_deallocating


调用objc_clear_deallocating函数。
```
void objc_clear_deallocating(id obj) 
{
    assert(obj);

    if (obj->isTaggedPointer()) return;
    obj->clearDeallocating();
}
```
我们顺着`clearDeallocating_slow`->`objc_object::clearDeallocating_slow`->`weak_clear_no_lock`->`weak_entry_remove`->`weak_compact_maybe`->......方法太多，总结起来太费事了（总算知道为什么大家的文章对于weak的释放写的那么语焉不详）。
总结objc_clear_deallocating的作用
>
1、从weak表中获取废弃对象的地址为键值的记录
    2、将包含在记录中的所有附有 weak修饰符变量的地址，赋值为nil
  3、将weak表中该记录删除
      4、从引用计数表中删除废弃对象的地址为键值的记录

