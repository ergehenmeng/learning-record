> 为每个使用该变量的线程提供独立的变量副本
>
> 线程本地变量保存,与并发实际上并没关系
>
> 即:在线程中保存一个局部变量,在该线程执行过程,获取时一定能获取到上次设置的值(前提不进行remove或者设置为null)



#### get过程

```java
//获取保存的变量
   public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
          //查找保存的变量
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
     //线程中还没创建保存变量的Map或者没有找到,则直接初始化默认变量值并创建Map
        return setInitialValue();
    }

	//获取value
  private Entry getEntry(ThreadLocal<?> key) {
    //通过Hash获取指定的下标
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
      return e;
    else
      //没找到(hash冲突了)
      return getEntryAfterMiss(key, i, e);
  }

//通过开放定址法进行查找(存在hash时,查找该下标后一位的值进行判断)
  private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
      ThreadLocal<?> k = e.get();
      if (k == key)
        return e;
      if (k == null)
        //发现有Entry不为空,key为空的节点,可能key已经被垃圾回收了,值还没被回收,因此需要清除
        expungeStaleEntry(i);
      else
        //当前下标的下一位
        i = nextIndex(i, len);
      e = tab[i];
    }
    return null;
  }

//清除staleSlot及其节点后key=null的Entry,同时返回最近一个Entry为空的节点的下标
  private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    //先把当前给清了
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;  
    Entry e;
    int i;
    //依次向后查找,如果发现key还有为空的,依旧清除
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
      ThreadLocal<?> k = e.get();
      if (k == null) {
        e.value = null;
        tab[i] = null;
        size--;
      } else {
        int h = k.threadLocalHashCode & (len - 1);
        //按正常来说,k在该位置但通过hash判断不是在该位置,说明k元素也是存在hash冲突被移过来的
        //将其归为到原来应该待的地方(因为这个位置可能已经腾出来了),如果已经存在冲突,则依次向后查找一个空位置放进去
        if (h != i) {
          tab[i] = null;
          while (tab[h] != null)
            h = nextIndex(h, len);
          tab[h] = e;
        }
      }
    }
    return i;
  }
```

#### set过程

```java
  private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
		//先进行hash冲突判断
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
      ThreadLocal<?> k = e.get();
      if (k == key) {
        e.value = value;
        return;
      }
			//该位置虽然被人占坑,但是key已经过期,可以替换为新的key和value
      if (k == null) {
        replaceStaleEntry(key, value, i);
        return;
      }
    }
		//没有hash冲突或一直存在hash冲突(即上述过程没有成功)
    tab[i] = new Entry(key, value);
    int sz = ++size;
    //新增或删除时,会重新清理过期的数据
    //该处表示没有清除掉过期的数据(所有数据都有用),则会进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
      rehash();
  }


//替换元素操作,同时尽可能的清理掉已经"过期"的数据
  private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                         int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;
    //应该放在staleSlot位置,但是先向前操作一波,看看是否还有key=null的Entry节点
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len)) {
      if (e.get() == null) {
         slotToExpunge = i;
      }
    }
  
    // 向后查找,虽然要放在staleSlot位置,但是前面也仅仅判断该位置为null(这个位置可能是其他元素先占的,只是后面被清除了而已)
    // 因此需要向后查找,看看是否已经真的存在当前key的元素
    for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
      ThreadLocal<?> k = e.get();
      //发现后面确实存在
      if (k == key) {
        e.value = value;
        //调换位置 注意tab[staleSlot]是个key为null的不为空的Entry
        tab[i] = tab[staleSlot];
        tab[staleSlot] = e;
		//此处表示在staleSlot节点前并没发现key=null的Entry节点存在,且staleSlot~i之间也没有发现key=null的Entry节点(因为有的话,下面逻辑会导致slotToExpunge值变更)
        if (slotToExpunge == staleSlot) {
          //目前也只是遍历(0-slotToExpunge[都不为空])
          //同样i-len之间依旧可能存在key=null的Entry存在
           slotToExpunge = i;
        }
        //expungeStaleEntry清除从i到len之间key=null的Entry节点的数据
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        return;
      }
      //当前节点为空,且(0-slotToExpunge)全部有值,记录第一次出现key=null的Entry节点的位置,方便清除
      if (k == null && slotToExpunge == staleSlot)
        slotToExpunge = i;
    }
	//自始至终没找到后面与之相同key
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);
	//但是在遍历的过程中找到了key=null的Entry节点,清除操作
    if (slotToExpunge != staleSlot)
      cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
  }

//尝试遍历从i-n之间的Entry节点,发现"过期"的entry则删除
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
      i = nextIndex(i, len);
      Entry e = tab[i];
      //发现有过期的元素,则直接从len/2处向后遍历
      if (e != null && e.get() == null) {
        n = len;
        removed = true;
        i = expungeStaleEntry(i);
      }
      //此处个人感觉减少无用的遍历
    } while ( (n >>>= 1) != 0);
    return removed;
  }
```

#### remove过程

```java

private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
      if (e.get() == key) {
        e.clear();
        //清除自身的同时也会将i之后的某些key=null的Entry清除了
        expungeStaleEntry(i);
        return;
      }
    }
  }

```

#### expungeStaleEntry与cleanSomeSlots总结

* `expungeStaleEntry` 遍历  从指定下标x到最近一个Entry(下标为y) 即 (x-y)之间所有Entry是否存在key=null的元素,如果存在则清除,注意 y的下标不是人为可控的(只要发现有Entry为空则暂停后续的清除操作)
* `cleanSomeSlots` 清除(x-n)节点所有key=null的Entry节点的值,n的位置是可控的



#### 内存溢出问题(个人看法)

> 由于`ThreadLocalMap`中Entry的key是弱引用,而key=ThreadLocal,在一般情况下 我们定义一个ThreadLocal都是staitc final的(官方也这么建议),因此key=null的可能性几乎为零,弱引用本身的作用无法提现出来,key和value都是强引用
>
> 内存溢出主要是在线程池中,我们一般使用Tomcat最为web容器,而Tomcat接收请求后交给线程池来处理我们的业务请求,因此线程无法被销毁,线程变量永远不会清空会造成内存泄露,尤其是项目中大规模使用ThreadLocal或者存储过多数据时
>
> 尽管set get 操作会在一定情况下清除key=null value不为空的数据,但是由于我们经常是用final,很少存在key=null的情况
>
> 因此强制建议在使用完线程变量后调用remove()方法进行清除