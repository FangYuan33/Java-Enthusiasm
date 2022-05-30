大家好，我是`方圆`，这篇博客儿是我参考了很多关于Java泛型上下界知识才完稿的，所以看完这篇，就真的没什么了！

---
## 1. 准备工作
- 有如下类的`继承关系`，为下文理解做好准备
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/a59bae00f4a643d2a82a4bbbc919d96d.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70#pic_center)
---

## 2. 没有泛型上下界遇到了什么问题？

`Apple是Fruit的子类`，那么如下代码是不会出错的
```java
Fruit apple = new Apple(); // ok
```
如果我们写如下代码，定义一个装有Fruit的List，并将装有Apple的List赋值给它，会如何呢？

```java
List<Fruit> plate = new ArrayList<Apple>();
```
它会在Idea里报红线，运行会报错：`java: 不兼容的类型: java.util.ArrayList<Apple>无法转换为java.util.List<Fruit>`，显然在集合间不存在继承引用关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/7175c08677de4f48bfe83337f9679e09.png)
那么面对以上问题，就需要上下界登场

---
## 3. 泛型的上界， ? extends T
使用泛型的上界，`? extends Fruit`，就能解决如上问题
```java
List<? extends Fruit> plate = new ArrayList<Apple>();
```
- 那么？我们该如何理解上界？
  ? 是java的通配符，在如上的例子中，上界`？ extends Fruit`代表任何继承了Fruit的子类（包含Fruit本身），该集合中装的就是这些元素
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2d5955d7781643019ecc1d6c8f12ba21.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70#pic_center)

- 那上界有什么`特点`？
  `只能取，不能存`
  先说只能取，这个很好理解，集合中都是Fruit的子类，那么取出来的`元素必然都能用Fruit来进行引用`，看如下代码很好理解
```java
 List<Apple> appleList = new ArrayList<>();
 appleList.add(new Apple());
 appleList.add(new Apple());
 
 List<? extends Fruit> plate = appleList;
 Fruit fruit = plate.get(0);
```
不能存又该如何理解呢？你不是说了列表里边都是Fruit的子类了吗？那我们向里边`随便加入它的子类`，怎么就不行呢？

听我慢慢道来！
![在这里插入图片描述](https://img-blog.csdnimg.cn/3cd4f48723a94b2498335ccd06f072bd.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
这段代码报了红线，确实不让添加，显示的错误如下图所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/bc5537cc76964d058d8375a5b8a35098.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
```java
List<Apple> appleList = new ArrayList<>(); // 泛型为Apple
List<? extends Fruit> plate = appleList;
```
那我们该先下来想一想，现在`plate`引用的是这个appleList，而appleList中泛型为`Apple`，那么也就意味着`plate的泛型也该是Apple`，我们向其中`添加Apple元素没有问题`，`添加RedApple和GreenApple都没有问题`，它们都能安全的转为Apple类型

![在这里插入图片描述](https://img-blog.csdnimg.cn/31679d7e307d40259d8d85da21b00e6e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)

但是！回过头来，我们再看，`? extends Fruit`，它表示的是任何继承了Fruit的子类，没问题吧，`照这么说添加Banana应该也可以啊`，Banana也是Fruit的子类啊，但是不行！别忘了，`我们已经将plate的指向到了appList，而appleList规定了泛型是Apple`，Banana向其中添加，它能转换成Apple吗？显然不能啊，但是它们确实都是Fruit的子类啊，也就是说，为了保证`绝对安全`，所以就限制了上界不能进行添加，以免发生转型失败的问题

![在这里插入图片描述](https://img-blog.csdnimg.cn/f4276b8ad99941dd81f31e50e16cb932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)

还要说一种比较好玩的情况

```java
 List<? extends Fruit> plate = Arrays.asList(new Apple(), new Banana());

 Fruit apple = plate.get(0);
 Fruit banana = plate.get(1);
```
在这种情况下，`这个plate真的成了能装任何水果的盘子`，里边有Apple和Banana，取出来之后确实都是Fruit
## 4. 泛型的下界， ? super T
- 泛型的下界以`? super Fruit`为例，它代表的就是任何Fruit的父类，包括了Fruit本身（还有Object哦）

![在这里插入图片描述](https://img-blog.csdnimg.cn/d0e6c29d67524bf697545765531a20d9.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70#pic_center)
- 那下界有啥子特点呐？`能存，其实也能取`，为什么说其实也能取呢，因为我看了一些文章，为了区分上下界，让它们的特点完全相反，都把下界的特点都写成了不能取，其实在代码中实践，能取出来，只不过会使其中的元素类型失效，取出来的元素类型都是Object

-  先来解释`能存`，看如下代码，注意它的`下界是Apple`哦

```java
// 定义了一个plate，它的下界为 ? super Apple
List<? super Apple> plate = new ArrayList<>();

plate.add(new Apple());
plate.add(new RedApple());
plate.add(new GreenApple());
```
这段代码在编译器里是没有问题，没有报错，正常运行
那有同学就有问题了，下界规定的是任何这个类型的父类，那`向其中添加Apple的父类为什么不行？`不是说好了可以能向其中存元素的吗？！怎么还报红线了！
![在这里插入图片描述](https://img-blog.csdnimg.cn/997e3420c42c430e8658e29edd2548c5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)

![等一下！这里](https://img-blog.csdnimg.cn/0609085406804fc4bde7d09c70e54f69.png)
我们先思考思考，上文提到的`绝对安全`，会不会还出现上文中转型失败的情况？
我们看一下? super Aplle的范围，如下图

![在这里插入图片描述](https://img-blog.csdnimg.cn/5f011943338045e2a8b10e4da2bdf1a6.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70#pic_center)

ok，添加Apple没问题，添加Fruit和Food也没问题，都在下界的范围内，但是，`谁能保证你就是添加的这几个类型的元素呢`？谁能来保证它的绝对安全呢？不能吧？！我向其中添加Banana，它肯定会出问题啊！

![在这里插入图片描述](https://img-blog.csdnimg.cn/4be310be29164641aa708c6dfbc05002.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)

那为什么又让添加Apple及其子类呢，因为它`绝对安全`，这些都可以安全的转型成Apple类啊，根本不会出啥毛病，向上转型完全不会出问题，所以是可以添加的，`下界的能存元素`是这个体现

- 再简单说一下什么叫`其实也能取`
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/cf71a27f95c54d8b990deadec48cd9b6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
  不过取出来的东西都是`Object`，使其中元素的类型失效了，而且因为`? super Apple`这个范围太大了，不知道拿出来是具体什么类型，只能用最高的Object来进行引用了

---

## 5. 它能进行怎样的应用？什么是PECS原则？
- 我们定义一个`MyStack`，如下，并添加了一个`pushAll方法`，将传入进来的List集合中的元素全部都压入栈中，但是值得注意的是，参数`List<E> fruits`没有使用上下界

```java
public class MyStack<E> extends Stack<E> {
    public void pushAll(List<E> fruits) {
        for (E fruit : fruits) {
            push(fruit);
        }
    }
}
```
我们写如下代码后，会有报错，`java: 不兼容的类型: java.util.List<Apple>无法转换为java.util.List<Fruit>`，和我们最初的问题是一样的，`List<Fruit> 不能引用 List<Apple>`，那我们将其改成`上界`，就能解决这个问题
![在这里插入图片描述](https://img-blog.csdnimg.cn/a769638d73a848c5abf9dacbf111a9c9.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
如下

```java
public class MyStack<E> extends Stack<E> {
    public void pushAll(List<? extends E> fruits) {
        for (E fruit : fruits) {
            push(fruit);
        }
    }
}
```
代码问题消失且能正常运行了，也能够将栈中的元素正常取出来
![在这里插入图片描述](https://img-blog.csdnimg.cn/81c83a012a1c4d44a323f550d63932d1.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
- 有了pushAll方法，也要写一个对应的`popAll方法`，如下，并没有规定`下界`哦

```java
public void popAll(List<E> fruits) {
    while (!isEmpty()) {
        fruits.add(pop());
    }
}
```
我们写如下代码，想将之前压入栈中的两个Apple`拿出来，并放入List<Food>这个集合中`，注意这里放入的是Fruit的父类Food，同样还是报了错误，`java: 不兼容的类型: java.util.List<Food>无法转换为java.util.List<Fruit>`
![在这里插入图片描述](https://img-blog.csdnimg.cn/57f062c34001497f99c6310deeb8aea6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
我们将代码进行修改，添加上`下界`之后，代码不报错且能正常运行了

```java
public void popAll(List<? super E> fruits) {
    while (!isEmpty()) {
        fruits.add(pop());
    }
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/acae5d6bd8074f2494bcd36d76ef3544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)

- PECS原则
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/760716fcec1d4de09700a8ff6a9ce561.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ2MjI1ODg2,size_16,color_FFFFFF,t_70)
- 
- 该如何理解生产者和消费者呢，`? extends E`一直从本身中拿元素出来，并将这些拿出来的元素供栈使用，我们将其成为`生产者`；`? super E`则相反，它是一直向集合中进行添加，就好像给它的东西都被它消费了一样，所以我们称它为`消费者`

那什么是PECS原则？

`Producer Extends，Consumer Super`，也就是说，如果一个参数类型是`生产者`的话，我们将采用`? extends T`上界，如果一个参数类型是`消费者`的话，那么就采用的是`? super T`下界


---
## 巨人的肩膀
- [Java 泛型 <? super T> 中 super 怎么 理解？与 extends 有何不同？](https://www.zhihu.com/question/20400700)
- [PECS法则与extends和super关键字](https://www.cnblogs.com/dldrjyy13102/p/8297045.html)
- 《Effective Java（第三版）》 第31条
- [JAVA通配符——? extends T ，? super T（尖括号不让打吗不是吧不是吧）](https://blog.csdn.net/qq_34025034/article/details/106562252)