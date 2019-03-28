---
layout: post
title: "垃圾回收的算法与实现读书笔记-引用计数法"
date: 2019-03-26 17:19:43 +0800
comments: true
categories: ["垃圾回收", "GC", "引用计数法"]
---

在[上一篇文章](http://linuxlsx.top/blog/2017/12/28/la-ji-hui-shou-de-suan-fa-yu-shi-xian-du-shu-bi-ji-biao-ji-zheng-li-suan-fa/)中介绍了`标记-清除法`的实现，这边接着来讲`引用计数法`的实现

文中部分图片引用自《垃圾回收的算法与实现》一书，如有版权问题，请联系删除。

# 引用计数法

`引用计数法`是George E.Colins 在1960年提出来的。主要思想是:

```C
让每个对象都记录下被多个其他对象引用的数量，当没有其他对象引用时，就可以让该对象被回收。
```

引用计数算法理解起来比较简单。首先来看下伪代码

<!-- more -->

## 伪代码

```C
//分配对象
new_obj(size){

    //向堆请求内存分配
    obj = pickup_chunk(size, $free_list)

    if(obj == NULL)
        //如果请求失败的话，直接失败
        allocation_fail()
    else
        //设置初始的引用计数为 1
        obj.ref_cnt = 1
        return obj
}


//引用变化时更新引用计数 ptr->obj
update_ptr(ptr, obj){
    //增加obj的计数
    inc_ref_cnt(obj)
    //减少原对象的计数
    dec_ref_cnt(*ptr)
    //更新引用
    *ptr = obj
}

//增加对象的引用计数
inc_ref_cnt(obj){
    obj.ref_cnt++
}

//减少对象的引用计数，在这里进行回收
def_ref_cnt(obj){
    obj.ref_cnt--

    if(obj.ref_cnt == 0)
        //当对象的引用计数为0时，遍历所有引用的对象，并将它们的计数减 1
        for (child : children(obj))
            def_ref_cnt(*child)

        //释放对象，回收内存
        reclaim(obj)
    }
}
```

更新对象计数时需要先增加目标对象的计数，然后再减少源对象的计数器。这样是为了防止目标对象和源对象是同一个对象时，先减少源对象计数器可能会导致对象计数器归零，被当成垃圾回收掉。

## 数据结构

为了实现引用计数法，需要在对象头中增加一个`引用计数器`。新的数据结构为：

```Java
public class ReferenceCountObj extends Obj {

    /**
     * 表示该对象的引用计数
     */
    public int count;

}
```

`Obj`对象的定义见[标记-清除法](http://linuxlsx.top/blog/2017/12/28/la-ji-hui-shou-de-suan-fa-yu-shi-xian-du-shu-bi-ji-biao-ji-zheng-li-suan-fa/)

## 算法实现

引用计数算法中主要的处理在`update_ptr`函数。请看下图，初始状态从根引用**A**和**C**，从**A**引用**B**。**A**持有指向**B**的唯一指针，假设现在将该指针更新到**C**。

![update_ptr](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/update_ptr%E5%87%BD%E6%95%B0%E5%AE%9E%E7%8E%B0.png?x-oss-process=style/black
)

通过以上的更新，**B**的计数器值变成了**0**，因此**B**被回收了，且**B**连上了空闲列表，可以被重新利用。又因为新形成了由**A**指向**C**的指针，所以**C**的计数器的值增为**2**。

以下是具体的一个实现代码:

```Java
public class ReferenceCountAlgo {

    protected Heap heap;

    public ReferenceCountAlgo(int size, int fitStrategy) {
        heap = new Heap(size, fitStrategy);
    }

    /**
     * 表示GC 的根
     */
    ReferenceCountObj root = new ReferenceCountObj();

    /**
     * 表示已经分配过的对象列表
     */
    protected LinkedList<ReferenceCountObj> allocatedObjList = new LinkedList<>();


    /**
     * 创建一个新的对象
     *
     * @param size 对象的大小
     * @return 创建好的对象
     * @throws OutOfMemoryError 当内存无法满足分配要求时抛出
     */
    public ReferenceCountObj newObj(int size) {

        Slot slot = heap.applySlot(size);

        //对于引用计数法来说，如果slot 为空，说明内存已经耗尽了
        if (slot == null) {
            throw new OutOfMemoryError("------ oh, out of memory! ------");
        }

        ReferenceCountObj obj = initObj(slot);
        allocatedObjList.add(obj);
        return obj;
    }

    /**
     * 建立对象之间的引用关系。 相当于 from = to
     * @param from      引用的起始对象。
     * @param to        应用的目标对象。该对象的引用计数需要加一
     */
    public void reference(ReferenceCountObj from, ReferenceCountObj to) {
        incrRefCount(to);
        from.children.add(to);
    }

    /**
     * 删除对象之间的引用关系。相当于 from = null
     * @param from    引用 '需要解除引用对象' 的对象。 如果 from == null, 说明是要解除和根的引用
     * @param to      需要解除引用的对象。该对象的引用计数会减一
     */
    public void deReference(ReferenceCountObj from, ReferenceCountObj to){

        if(from == null){
            root.children.remove(to);
        }else {
            from.children.remove(to);
        }

        defRefCount(to);
    }

    /**
     * 修改对象的引用关系。相当于从 from = oldTo 变化为 from = newTo
     * @param from
     * @param oldTo
     * @param newTo
     */
    public void changeReference(ReferenceCountObj from, ReferenceCountObj oldTo, ReferenceCountObj newTo){

        from.children.remove(oldTo);
        defRefCount(oldTo);
        from.children.add(newTo);
        incrRefCount(newTo);

    }

    private void incrRefCount(ReferenceCountObj obj) {
        obj.count++;
    }

    private void defRefCount(ReferenceCountObj obj) {
        obj.count--;

        //如果对象的计数变为0，则直接进行回收操作
        if (obj.count == 0) {

            //循环遍历该对象引用的对象，对其进行引用减一操作
            for (Obj child : obj.children) {
                ReferenceCountObj countObj = (ReferenceCountObj) child;
                defRefCount(countObj);
            }

            //释放掉内存
            heap.release(new Slot(obj.start, obj.size));
        }
    }

    private ReferenceCountObj initObj(Slot slot) {
        ReferenceCountObj obj = new ReferenceCountObj();

        obj.start = slot.start;
        obj.size = slot.size;
        //对象的初始化引用计数为 0
        obj.count = 0;

        return obj;
    }

    /**
     * 把对象置为根节点
     *
     * @param obj 根对象
     */
    public void makeItToRoot(ReferenceCountObj obj) {
        root.children.add(obj);
        obj.count++;
    }
}
```

### 引用计数法的优点

    * 当对象的计数器归零后可立刻回收
    * 最大暂停时间短。在操作对象的时候同时进行垃圾回收，不需要专门的阶段来进行回收
    * 回收只需要考虑当前对象及其所以引用的对象，不需要从根对象开始

### 引用计数法的缺点

    * 循环引用无法回收。
    * 计数器会带来额外的读/写开销，频繁的写操作可能会导致高速缓存失效。所以引用计数法不适合通用的大容量、高性能的内存管理场景。
    * 用于计数的计数器必须能够满足一个对象引用堆中所有其他对象的极端情况。所有计数器大小必须和引用类型一样大，会造成内存浪费。
    * 在多线程环境中需要保证引用计数的增减以及加载和存储指针的操作都必须原子化。

## 算法优化

    1. 降低计数器带来的开销
       1.1 延迟计数法。将应用计数延迟某个阶段统一的执行
       1.2 合并计数法。不在每次变更都更新计数器，而是关注开始和结束状态，对中间的变更进行合并。
    2. 解决循环引用无法回收的问题
       2.1 通过部分标记的方法解决循环引用的问题。

### 延迟计数法

在`延迟计数法`中，引入了一个新的结构ZCT(Zero Count Table)，来保存计数器归零的对象。相比原来的实现，有以下几个步骤需要进行调整。

**1. 创建对象** 当对象内存分配失败时，要先遍历下ZCT释放内存，然后再次尝试申请内存，如果还是失败的话，就真的是内存不足了。

```Java
    @Override
    public ReferenceCountObj newObj(int size) {
        Slot slot = heap.applySlot(size);
        //对于引用计数法来说，如果slot 为空
        //不直接抛出OOM，而是释放掉ZCT中可以释放的对象
        if (slot == null) {
            scanZctToReleaseMemory();
            slot = heap.applySlot(size);
            if (slot == null) {
                throw new OutOfMemoryError("------ oh, out of memory! ------");
            }
        }
        ReferenceCountObj obj = initObj(slot);
        allocatedObjList.add(obj);

        return obj;
    }
```

**2. 计数器操作** 当对象计数器的值降到`0`时，不要直接将对象回收，而是将对象添加到ZCT中。另外为了减少从根节点应用的对象的计数器频繁变更，规定当应用是来自根节点的时候，计数器不加`1`。

```Java
   /**
     * 模拟和根节点建立引用关系
     * @param obj 根对象
     */
    @Override
    public void makeItToRoot(ReferenceCountObj obj) {
        //如果对象的应用是来自根节点，那么就不要修改对象的计数器
        root.children.add(obj);

        //因为对象的计数器为0，所以将其放到ZCT中
        if (isFull()) {
            scanZctToReleaseMemory();
        }
        zct.add(obj);
    }

    @Override
    public void deReference(ReferenceCountObj from, ReferenceCountObj to) {

        //因为根节点引用的对象初始计数器为0，所以释放的时候不需要操作计数器
        if(from == null){
            root.children.remove(to);
        }else {
            from.children.remove(to);
            defRefCount(to);
        }
    }

    /**
     * 与原始实现不同，当计数器变成0时，将对象放到ZCT中而不是直接释放
     * @param obj
     */
    @Override
    protected void defRefCount(ReferenceCountObj obj) {
        obj.count--;

        //如果对象的计数变为0，则优先把对象放到zct中
        if (obj.count == 0) {
            //如果队列满了，则释放掉队列中的对象
            if (isFull()) {
                scanZctToReleaseMemory();
            }

            zct.add(obj);
        }
    }

```

引入ZCT后，根节点引用的对象和计数器为0的对象同时存在ZCT中，如下图所示：

![ZCT](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/zct%E7%A4%BA%E6%84%8F%E5%9B%BE.png?x-oss-process=style/black)

**3. 新引入的函数 scanZctToReleaseMemory** 在这个函数中会遍历ZCT，释放内存。

```Java
    private void scanZctToReleaseMemory() {

        //在进行释放操作之前要对根节点引用的对象做计数器加1的动作，防止对象被误回收
        for (Obj child : root.children) {
            ((ReferenceCountObj)child).count++;
        }

        Iterator<Obj> iterator = zct.iterator();
        while (iterator.hasNext()) {

            ReferenceCountObj countObj = (ReferenceCountObj) iterator.next();
            //对于计数器为0的对象，从ZCT中移除
            iterator.remove();
            //并删除该对象回收内存
            if (countObj.count == 0) {
                delete(countObj);
            }
        }

        //在进行释放操作之后要对根节点引用的对象做计数器减1的动作
        //同时将计数器为0的对象也放到 ZCT中
        for (Obj child : root.children) {
            ReferenceCountObj obj = (ReferenceCountObj) child;
            obj.count--;

            if(obj.count == 0){
                zct.add(obj);
            }

        }
    }

    private void delete(ReferenceCountObj countObj){

        //释放内存
        releaseMemory(countObj);

        //减少引用对象的计数器值，如果计数器值为0，回收之
        for (Obj child : countObj.children) {
            ReferenceCountObj childCountObj = (ReferenceCountObj) child;
            childCountObj.count--;
            if(childCountObj.count == 0){
                delete(countObj);
            }
        }
    }
```

以下是一个简单的对比，可以看到延迟计数法在对象计数器归零时并没有立刻回收，而原实现是立刻回收的。

![效果分析](
http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/defered%E6%95%88%E6%9E%9C%E5%88%86%E6%9E%90.png?x-oss-process=style/origin
)

优点：延迟计数法的有点是降低了引用频繁变化带来的计数器开销。

缺点：延迟计数法使得垃圾不能立刻回收，同时在多线程环境中需要引入额外的同步机制保证内存清理的正确性。

### 部分标记解决循环引用

`部分标记`的意思是针对部分有可能是循环引用垃圾的对象进行标记。其主要的依据是：

* 在循环引用垃圾内部，所有对象的应用计数均由其内部对象之间的指针产生。
* 只有再删除某一对象的某个引用后该对象的引用计数仍大于零时，才有可能出现循环引用垃圾。

部分标记使用四种不同的颜色来标记对象不同的状态：

1. 黑色(BLACK): 绝对不是垃圾的对象
2. 白色(WHITE): 绝对是垃圾对象
3. 灰色(GRAY): 搜索完毕的对象
4. 阴影(HATCH): 可能是循环垃圾的对象

部分标记对一个可能的循环引用垃圾进行子图追踪。对于每一个遍历到的每个引用，算法对其目标对象进行试验删除，临时性的减少目标对象的引用计数，从而移除由内部引用产生的计数。追踪完成以后，如果某个对象的计数仍然大于零，则必然是因为循环以外的其他对象引用了该对象，从而判断该对象及其循环引用都不是垃圾。

部分标记的逻辑比较复杂，结合图示例来讲会比较清楚一些。下图是一个引用结构的初始状态，有循环引用的对象群是**ABC**和**DE**,其中**A**和**D**由根引用。此外，**C**和**E**引用到**F**。

![初始状态](
http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-%E5%88%9D%E5%A7%8B%E7%8A%B6%E6%80%81.png?x-oss-process=style/black)

现在我们删除从根到A对象的引用，逻辑看代码实现：

```Java
    protected void defRefCount(ReferenceCountObj obj) {
        obj.count--;

        //如果对象的计数变为0，则直接进行回收操作
        if (obj.count == 0) {

            //循环遍历该对象引用的对象，对其进行引用减一操作
            for (Obj child : obj.children) {
                ReferenceCountObj countObj = (ReferenceCountObj) child;
                defRefCount(countObj);
            }
            releaseMemory(obj);
        } else if (obj.color != Color.HATCH) {
            //对象的颜色不为 HATCH的时候，将其颜色标记为 HATCH，然后加入到队列中
            obj.color = Color.HATCH;
            hatchQueue.add(obj);
        }
    }
```

**A**的引用计数减少后的状态如下图，因为**A**的引用计数不为零，所以是疑似的循环垃圾，被标记成了`HATCH`。
![A](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-%E7%A7%BB%E9%99%A4%E6%A0%B9%E5%88%B0A.png?x-oss-process=style/black)

引用计数触发回收的入口都在创建对象的时候，所以创建对象的代码也需要进行修改：

```Java
    public ReferenceCountObj newObj(int size) {
        Slot slot = heap.applySlot(size);
        if (slot != null) {

            ReferenceCountObj obj = initObj(slot);
            //新对象的颜色置为黑色
            obj.color = Color.BLACK;
            allocatedObjList.add(obj);
            return obj;
        } else if (!hatchQueue.isEmpty()) {
            //如果第一次分配失败，就尝试清理疑似垃圾
            scanHatchQueue();
            return newObj(size);
        }
        throw new OutOfMemoryError("------ oh, out of memory! ------");
    }
```

`scanHatchQueue`函数会执行可以循环垃圾的子图遍历。

```Java
    public void scanHatchQueue(){
        ReferenceCountObj obj = hatchQueue.poll();
        if(obj.color == Color.HATCH){
            //通过以下三步找出循环垃圾并回收之
            paintGray(obj);
            scanGray(obj);
            collectWhite(obj);
        }else if(!hatchQueue.isEmpty()){
            scanHatchQueue();
        }
    }
```

`paintGray`函数会对目标对象进行试验删除，临时性的减少目标对象的引用计数，从而移除由内部引用产生的计数

```Java
    /**
     * 把对象的颜色置为 GRAY，对子对象计数器减1，递归调用 paintGray。
     * @param obj
     */
    public void paintGray(ReferenceCountObj obj){
        if(obj.color == Color.BLACK || obj.color == Color.HATCH){
            obj.color = Color.GRAY;

            for (Obj child : obj.children) {
                ReferenceCountObj c = (ReferenceCountObj) child;
                // 这个地方非常重要，是子对象的计数器减1，而不是父对象。
                // 否则会造成错误回收的情况。
                c.count--;
                paintGray(c);
            }
        }
    }
```

对A及其子应用执行完`paintGray`后的状态。可以看到A,B,C,F都被标记成了灰色，A,B,C的计数为0，F的计数为1

![paintGray](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-%E6%89%AB%E6%8F%8FA.png?x-oss-process=style/black)

下面详细的描述下过程:

![详细过程](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-%E5%BE%AA%E7%8E%AF%E6%A0%87%E8%AE%B0%E5%90%8E.png?x-oss-process=style/black)

首先，在(a)中A被涂成了灰色，程序对B进行了计数器减量操作。在(b)中B也被涂成了灰色，对C进行了减量操作。在(c)中C被涂成了灰色，对A,F进行了计数器减量的操作。

接下来开始对灰色对象进行扫描。

```Java
    public void scanGray(ReferenceCountObj obj){

        if(obj.color == Color.GRAY){
            if(obj.count > 0){
                //如果经过paintGray操作后，引用计数还是大于零，
                //对象肯定有来自其他对象的引用，则将对象的颜色标记为黑色
                paintBlack(obj);
            }else {
                //对计数为零的对象标记为白色。
                obj.color = Color.WHITE;
                for (Obj child : obj.children) {
                    ReferenceCountObj c = (ReferenceCountObj) child;
                    scanGray(c);
                }
            }
        }
    }
```

执行完`scanGray`之后的状态，A,B,C都被标记成了白色，F因为有来自E的引用，所以重新被标记成了黑色。

![scanGray](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-scanGray.png?x-oss-process=style/black)

最后就是对标记为白色对象进行回收了。

```Java
    public void collectWhite(ReferenceCountObj obj){
        if(obj.color == Color.WHITE){
            obj.color = Color.BLACK;

            for (Obj child : obj.children) {
                ReferenceCountObj c = (ReferenceCountObj) child;
                collectWhite(c);
            }

            releaseMemory(obj);
        }
    }
```

回收的状态如下图，至此循环垃圾已经被回收完成了。

![回收完成](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-collectWhite.png?x-oss-process=style/black)

下图表示了在算法中四种颜色的变化示意图:

![颜色变化](http://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/%E9%83%A8%E5%88%86%E6%A0%87%E8%AE%B0-%E9%A2%9C%E8%89%B2%E5%8F%98%E5%8C%96.png?x-oss-process=style/black)

`部分标记法`可以解决循环垃圾的问题，但是在效率上也存在不足。当对象很多的情况下，遍历对象的成本会变高，会增加垃圾回收的暂停时间。

## 总结

引用计数法虽然思想比较简单，但是工程实现上比较困难。在大型系统上基本上没有用武之地，但是在大多数对象的生命周期简单到直接进行管理的混合场景会比较有效。