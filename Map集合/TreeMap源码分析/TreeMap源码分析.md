    ### TreeMap源码分析
    
    ##### 一、TreeMap的特性    
    
    1.key必须实现Comparable或者Comparator接口  
    2.底层是基于红黑树来实现的，时间复杂度都是log(n)  
    3.TreeMap的结点的默认顺序是Key的自然顺序(Key必须实现Comparator). 当然, TreeMap内部也支持外部提供Comparator来指定结点的排列顺序.   
    
    ***  
    ##### 二、源码分析  
    - put方法
    ```
     public V put(K key, V value) {
            Entry<K,V> t = root;
            if (t == null) {
                compare(key, key); // type (and possibly null) check
    
                root = new Entry<>(key, value, null);
                size = 1;
                modCount++;
                return null;    //root为空就初始化，并且直接返回
            }
            int cmp;
            Entry<K,V> parent;
            // split comparator and comparable paths
            Comparator<? super K> cpr = comparator;
            if (cpr != null) {  //comparator构造器不为空
                do {
                    parent = t; //parent为root节点
                    cmp = cpr.compare(key, t.key);  //比较compare方法
                    if (cmp < 0)
                        t = t.left; //key比root小，那么t = root的左边
                    else if (cmp > 0)
                        t = t.right;
                    else
                        return t.setValue(value);
                } while (t != null);
            }
            else {
                if (key == null)    //键为空，抛空指针异常
                    throw new NullPointerException();
                @SuppressWarnings("unchecked")
                    Comparable<? super K> k = (Comparable<? super K>) key;
                do {
                    parent = t;
                    cmp = k.compareTo(t.key);
                    if (cmp < 0)
                        t = t.left;
                    else if (cmp > 0)
                        t = t.right;
                    else
                        return t.setValue(value);   //key值相同直接覆盖
                } while (t != null);
            }   //上面这个du..while循环的作用就是，使用compareTo方法，大于0就放在Entry右边，否则放在左边
            Entry<K,V> e = new Entry<>(key, value, parent); //构造函数创建Entry
            if (cmp < 0)
                parent.left = e;    //为父节点创建子节点
            else
                parent.right = e;    //为父节点创建子节点
            fixAfterInsertion(e);   //红黑树的修复。。。这里我不想分析了，无非就是变色旋转
            size++;
            modCount++;
            return null;
        }
    ```  
    
    
    **上面的put方法我们就看到，key没实现comparable或者compator接口的话，会抛异常，并且key是不能为空，还有就是key值相同也是直接覆盖的**   
    
    
    - get方法  
    ```
      public V get(Object key) {
            Entry<K,V> p = getEntry(key);
            return (p==null ? null : p.value);
        }
    ```  
    
    所以重点是getEntry(key)方法  
    
    ```
    
      final Entry<K,V> getEntry(Object key) {
            // Offload comparator-based version for sake of performance
            if (comparator != null) //没传comparator的话，看看父类有没有comparator，并且使用父类的comparator进行查询
                return getEntryUsingComparator(key);    //里面的逻辑和下面的while是一样的
            if (key == null)
                throw new NullPointerException();   //key为空抛异常
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            Entry<K,V> p = root;
            while (p != null) {
                int cmp = k.compareTo(p.key);  
                if (cmp < 0)
                    p = p.left;
                else if (cmp > 0)
                    p = p.right;
                else
                    return p;
            }   //利用compareTo方法，小于0就往树的左边找，大于0就往树的右边找
            return null;
        }
    ```   
    
    
    **实现了Compartor或者Comparable就好了，利用while循环，遍历红黑树，就能找到结果**
    
