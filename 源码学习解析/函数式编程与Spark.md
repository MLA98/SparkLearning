## 开篇
第一篇先讲讲函数式编程在Spark中的应用，因为理解函数式编程对理解Spark的一些原理还是很有帮助的。

## 函数式编程简介
函数式编程有两个比较重要的点，一个是无副作用，一个是immutable变量。
1. 无副作用： 
```f(x)=x + 1```，对于这个函数，只要它的入参是n，它的输出值永远都是n+1。因为这个函数不依赖任何外部的状态，只依赖入参。这一特性是Spark的RDD懒加载的理论基础。
2. immutable变量：
这个特性指的是，所有变量都应该是不可变的，在Spark中，RDD就是不可变的。它和无副作用是Spark容错机制lineage的理论基础。

## Transform，action和懒加载（Lazy Loading）
编写过一些Spark程序的人，应该了解Spark有两种对RDD/dataset的操作。一种是Transform类型，包括map，mapPartition等；一种是action类型，包括count，reduce等。

看下这个例子：
```
val sampleRdd = sc.makeRDD(Array(1, 2, 3, 4), 2)
val rdd2 = sampleRdd.filter((str) => str >= 2)
val count = rdd2.count()
```
spark会等到调用count()的时候才执行filter(),这就是懒加载，只有用户真正需要结果的时候才执行全过程。所有transform类型的操作都是懒加载的，而action类型会触发rdd上的所有transform。懒加载可以让程序集中运行，节约资源，而且在Spark这样的分布式程序里可以节约大量的通信调度时间。

为什么函数式编程的无副作用的对懒加载相当重要呢，我们来看下面这个例子。
```
val xMinBefore = (x : Int) => {
    System.currentTimeMillis / 1000 - 60 * x;
}
```
看到这个函数，它就是有副作用的，它的运行结果不仅依赖入参，还依赖于这个世界的时间，所以你在不同时间去执行这个函数，返回的值是不同的。对于这种有副作用的函数，我们自然不能使用懒加载，我们需要在调用的时候就得到结果，才能保证结果是我们需要的。而没有副作用的函数，我们可以在任何时候执行都得到一致的结果，自然可以在任何时候调用，也就保证了在最后加载的时候，输出的数据是不会有问题的。

## RDD的Lineage 
保证了无副作用，Spark的容错机制lineage才能起作用。回到上面那个例子
```
val sampleRdd = sc.makeRDD(Array(1, 2, 3, 4), 2)
val rdd2 = sampleRdd.filter((str) => str >= 2)
val count = rdd2.count()
```
rdd2是在sampleRdd的基础上进行了filter操作，rdd2实际上就是保留了一个对sampleRdd的引用，并记录了filter操作。rdd2这样就有了和samleRdd的lineage。

lineage的应用：
在执行第三行count()时，rdd2就会在sampleRdd的基础上执行filter操作。当执行filter的时候失败了，spark就会往上追溯到上一个成功的rdd，也就是sampleRdd，再执行一把。可以使用这种方式容错的前提，就是RDD的immutable特性以及无副作用，它们保证在任何时候对一个RDD执行相同操作时，输出的结果永远是一样的。