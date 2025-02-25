# List集合中容易出错的移除方法
结论：
- 正确搭配1：不推荐，通过索引遍历形如for int i  =0; i<n;i++ 搭配list.remove(index)/list.remove(obj)【只修改modCount】，但存在漏删的问题，要注意避免
- 正确搭配2：搭配2中不可将list.remove与iterator混用！！！，增强for循环本质也是迭代器，通过增强for循环/foreach/Iterator遍历+iterator.remove()【同时修改modCount和expectedModCount】，这样不会存在ConcurrentModificationException

## list.remove(index)错误分析

list内部会自动修改索引，元素前移同时长度减少，因此需要注意避免漏删的问题
```java
    /**
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
    private void fastRemove(Object[] es, int i) {
        modCount++;
        final int newSize;
        if ((newSize = size - 1) > i)
            System.arraycopy(es, i + 1, es, i, newSize - i);
        es[size = newSize] = null;
    }
```
另外，如果使用是get(i)，相当于用的是拷贝后的item，而不是直接使用当前元素的引用，
，因此在循环读取的同时删除元素并不会抛出下面的异常：java.util.ConcurrentModificationException
，只是会出现漏删问题而已

但是如果使用的是迭代器或者foreach直接取的元素引用，用list.remove(obj)进行删除，则会抛出异常：java.util.ConcurrentModificationException
，下面进行分析

## list.remove(obj)错误分析
抛出异常：java.util.ConcurrentModificationException

这正式由于上述list内部会自动修改索引的问题导致的

foreach 写法实际上是对的 Iterable、hasNext、next方法的简写。因此从List.iterator()源码着手分析，跟踪iterator()方法，该方法返回了 Itr 迭代器对象。
```java
public Iterator<E> iterator() {
    return new Itr();
}
```
Itr 类定义如下：

通过代码我们发现 Itr 是 ArrayList 中定义的一个私有内部类，在 iterator.next()、iterator.remove()方法中都会调用checkForComodification 方法，

checkForComodification方法的作用是判断 modCount != expectedModCount是否相等，如果不相等则抛出ConcurrentModificationException异常。

每次正常执行 iterator.remove() 方法后，都会对执行expectedModCount = modCount赋值，保证两个值相等。

iterator.remove():
```java
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;
 
        public boolean hasNext() {
            return cursor != size;
        }
 
        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
 
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
 
            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
 
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

错误的case：在 foreach/iterator 循环中执行 list.remove(obj);
```java
    /**
     * ConcurrentModificationException
     */
    public static void errIteratorRemove() {
        Iterator<Item> iterator = list.iterator();
        while (iterator.hasNext()){
            Item next = iterator.next();
            if (StringUtils.equals(next.getName(), "item1")) {
                list.remove(next);
            }
        }
        System.out.println(list);
    }
```
因为list.remove(obj) 仅会对 list 对象的 modCount 值进行了修改，不会对expectedModCount重新赋值，因此在后续的iterator.next()过程中，由于Itr#checkForComodification校验抛出了ConcurrentModificationException异常。
```java
        /**
     * Removes the first occurrence of the specified element from this list,
     * if it is present.  If the list does not contain the element, it is
     * unchanged.  More formally, removes the element with the lowest index
     * {@code i} such that
     * {@code Objects.equals(o, get(i))}
     * (if such an element exists).  Returns {@code true} if this list
     * contained the specified element (or equivalently, if this list
     * changed as a result of the call).
     *
     * @param o element to be removed from this list, if present
     * @return {@code true} if this list contained the specified element
     */
    public boolean remove(Object o) {
        final Object[] es = elementData;
        final int size = this.size;
        int i = 0;
        found: {
            if (o == null) {
                for (; i < size; i++)
                    if (es[i] == null)
                        break found;
            } else {
                for (; i < size; i++)
                    if (o.equals(es[i]))
                        break found;
            }
            return false;
        }
        fastRemove(es, i);
        return true;
    }
    /**
     * Private remove method that skips bounds checking and does not
     * return the value removed.
     */
    private void fastRemove(Object[] es, int i) {
        modCount++;
        final int newSize;
        if ((newSize = size - 1) > i)
            System.arraycopy(es, i + 1, es, i, newSize - i);
        es[size = newSize] = null;
    }
```