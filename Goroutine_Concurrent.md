#### 系统线程和Gorountine

系统级线程会有一个固定大小的栈（一般是2MB），主要用来保存函数递归调用时参数和局部变量。而Goroutine会根据需要动态地伸缩栈的大小，代价更小。

Go的运行时还包含了其自己的调度器，可以在n个操作系统线程上多工调度m个Goroutine。Goroutine采用的是半抢占式的协作调度，只有在当前Goroutine发生阻塞时才会导致调度；同时发生在用户态，调度器会根据具体函数只保存必要的寄存器，切换的代价要比系统线程低得多。用runtime.GOMAXPROCS控制当前运行正常非阻塞Goroutine的系统线程数目。

#### 顺序一致性内存模型


```
var a string
var done bool

func setup() {
    a = "hello, world"
    done = true
}

func main() {
    go setup()
    for !done {}
    print(a)
}
```

Go语言并不保证在main函数中观测到的对done的写入操作发生在对字符串a的写入的操作之后，因此程序很可能打印一个空字符串。更糟糕的是，因为两个线程之间没有同步事件，setup线程对done的写入操作甚至无法被main线程看到，main函数有可能陷入死循环中。

在Go语言中，同一个Goroutine线程内部，顺序一致性内存模型是得到保证的。但是不同的Goroutine之间，并不满足顺序一致性内存模型，需要通过明确定义的同步事件来作为同步的参考。如果两个事件不可排序，那么就说这两个事件是并发的。为了最大化并行，Go语言的编译器和处理器在不影响上述规定的前提下可能会对执行语句重新排序（CPU也会对一些指令进行乱序执行）。

因此，如果在一个Goroutine中顺序执行a = 1; b = 2;两个语句，虽然在当前的Goroutine中可以认为a = 1;语句先于b = 2;语句执行，但是在另一个Goroutine中b = 2;语句可能会先于a = 1;语句执行，甚至在另一个Goroutine中无法看到它们的变化（可能始终在寄存器中）。也就是说在另一个Goroutine看来, a = 1; b = 2;两个语句的执行顺序是不确定的。如果一个并发程序无法确定事件的顺序关系，那么程序的运行结果往往会有不确定的结果。

###### 备注
如果GOMAXPROCS设置为1，相当于是单线程Goroutine并发，main函数中Goroutine一直占用着cpu不释放，所以导致另一个Goroutine得不到执行, 进入死循环了。 第二个问题是不管GOMAXPROCS的设置情况， 在main函数Goroutine里面观察到a，done的赋值顺序是随机的，所以会出现打印a为""的情况。

#### 参考资料
1. [GO语言高级编程](https://chai2010.cn/advanced-go-programming-book/ch1-basic/ch1-05-mem.html)