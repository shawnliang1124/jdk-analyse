
##### 1.ArrayList的结构
![image](https://wx3.sinaimg.cn/mw1024/007R8l8Fgy1g9jhcgr2aoj30v706wwfc.jpg)  

我们这里注意一下ArrayList中的关键几个属性  

默认的容量大小  
DEFAULT_CAPACITY    //10  
 transient Object[] elementData    //空的obj数组  


那么我们直接从add方法进行分析吧。  
```   
    /**
     * Appends the specified element to the end of this list.   
     *将值的内容加到数组的最后一行里面
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  //size代表的是ArrayList的大小，add的时候就是自动+1,这个方法主要判断数组要不要进行扩容
        elementData[size++] = e;   //数组的最后，赋值 
        return true;
    }

```  

再调用ensureCapacityInternal(size+1)的方法  

```  
  private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);  //无参构造没有传大小，取空的Obj数组大小
        }

        ensureExplicitCapacity(minCapacity);
    }

```  


再调用ensureExplicitCapacity()方法进行判断    

```  
 private void ensureExplicitCapacity(int minCapacity) {
        modCount++;     //abstractList抽象类中定义的数组结构变化的次数+1

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);  //假设obj数组的大小小于传入的数字大小，进入扩容的方法
    }

```  

我们来关注一下grow，扩容的方法  
```  
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;   //老数组的大小
        int newCapacity = oldCapacity + (oldCapacity >> 1); //新数组大小 = 老数组大小+老数组大小/2 =  1.5倍老数组
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;  //新数组大小若小于传入的大小值，那么就令新数组大小直接=传入的大小值
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);    //新数组大小超过最大值，抛出OOM异常
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);  //调用copy方法，将旧数组的值copy到新数组里面去，copy方法的底层调用的是native方法
    }

```  

如果在需要扩容的情况下，新数组扩容的大小将是老数组的1.5倍，并且调用了==Arrays.copy==方法进行复制，将老数组的内容复制到新数组里面。 PS，自己点入Arrays.copy里面，底层用的也是System.arraycopy方法  
``` 
 public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        @SuppressWarnings("unchecked")
        T[] copy = ((Object)newType == (Object)Object[].class)
            ? (T[]) new Object[newLength]
            : (T[]) Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
```   


**总结：所以add(E e)方法也是比较简单的，首先判断数组的大小是否需要扩容，如果需要扩容了，那么就调用底层的 System.arraycopy方法进行扩容，最后在obj数组的最后一个位置加上新的值即可**   

***  
我们再来看看，add(int index,E element)的方法
```  
 public void add(int index, E element) {
        rangeCheckForAdd(index);    //判断索引下标是否合法

        ensureCapacityInternal(size + 1);  // 该方法扩容的时候已经复制了一次数组
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index); //这次还要copy一次数组的值，将index的值让出来
        elementData[index] = element;
        size++;
    }
```  

**这个方法估计大家都已经发现了吧，它的效率比add(E e)更低，为什么，假设要扩容的情况下， ensureCapacityInternal(size + 1)方法已经扩容了一次值，之后，依旧调用了  System.arraycopy(elementData, index, elementData, index + 1,
                         size - index)方法，这个方法又扩容了一次值，两次复制严重影响效率** 
                         

***

get方法   
```  
  public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```  

get方法没什么好说的，直接去找到的数组下标，获取就好了。  

***  

ArrayList底层实际上就是一个Obj数组，当它的大小阈值达到10的时候，会以1.5倍的大小进行扩容，扩容之后会把老的数组以copy的形式赋值到新的数组上，如果调用了add(E e,int index)方法，在扩容的基础上还要copy数组，赋值一次，严重影响效率，所以ArrayList比较适合查询而不是新增。




