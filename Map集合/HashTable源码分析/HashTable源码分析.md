HashTable和HashMap相比最大的区别就是线程安全和非线程安全的，其实底层逻辑也是非常简单。  

底层数据结构，就是一个EntryK,V数组，假设发生了hash冲突就是一个链表结构，并且底层的put和get都用了synchronized关键字进行修饰 。直接看代码吧，一次过就行  

```
  public synchronized V put(K key, V value) {     synchronized修饰方法，线程安全
         Make sure the value is not null
        if (value == null) {    value不可为空
            throw new NullPointerException();
        }

         Makes sure the key is not already in the hashtable.
        Entry, tab[] = table;   Entry数组
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;   获取index位置
        @SuppressWarnings(unchecked)
        EntryK,V entry = (EntryK,V)tab[index];
        for(; entry != null ; entry = entry.next) { 遍历链表，有key的值相同就覆盖
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old; 假设代替就返回旧的值
            }
        }

        addEntry(hash, key, value, index);  
        return null;
    }

```


 ##### addEntry(hash, key, value, index)方法如下：  
 ```
 
    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry, tab[] = table;
        if (count = threshold) {
             Rehash the table if the threshold is exceeded
            rehash();   大于阈值，就rehash

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

         Creates the new entry.
        @SuppressWarnings(unchecked)
        EntryK,V e = (EntryK,V) tab[index];
        tab[index] = new Entry(hash, key, value, e);
        count++;
    }
 
 ```
 
 #####  rehash方法
 ``` 
   protected void rehash() {
        int oldCapacity = table.length;
        Entry,[] oldMap = table;

         overflow-conscious code
        int newCapacity = (oldCapacity  1) + 1;   新容量 = 2老容量+1
        if (newCapacity - MAX_ARRAY_SIZE  0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                 Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry,[] newMap = new Entry,[newCapacity];

        modCount++;
        threshold = (int)Math.min(newCapacity  loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i--  0 ;) {
            for (EntryK,V old = (EntryK,V)oldMap[i] ; old != null ; ) { 链表的插入和hashmap1.7一样，也是用的头插方法
                EntryK,V e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (EntryK,V)newMap[index];
                newMap[index] = e;
            }
        }
    }
 ```  
 
 
 ##### get方法  
 
 ```
 public synchronized V get(Object key) {
        Entry, tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;   获得table下标
        for (Entry, e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;  hash冲突的情况下，遍历链表，找到对应的值
            }   
        }
        return null;
    }
 
 ```   
 get方法也是用的synchronized修饰，底层也无非是获得Entry数组下标，如果发生hash冲突，就遍历链表找到对应的值即可。
 
 

 
 
 HashTable的源码分析相对于HashMap来说简单很多，底层也是Entry数组+单向链表的结构，数组大小到了阈值就调用rehash方法进行扩容，扩容大小是2X老数组大小+1，并且如果hash冲突的话也是采用的头插链表法倒置复制链表  
 
 
 PS，头插链表意思是说，假设一个链表的结构是1 ，2 ，3。重新复制后，就是3 ，2 ，1   
 
 
 代码实现：  
 EntryK,V e = old;
 e.next = (EntryK,V)newMap[index];
 newMap[index] = e;
 
   
 
 
 

