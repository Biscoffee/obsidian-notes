# _Inbox

## 2026-05-14 23:33 GCD 同步执行主队列死锁分析
**来源**: 博客　**confidence**: 0.90

- 在主线程中调用同步执行 + 主队列会导致死锁：任务与当前任务互相等待。
- 在其他线程中调用同步执行 + 主队列不会死锁，任务在主线程顺序执行。
- 异步执行 + 主队列只在主线程中顺序执行任务，不开启新线程。
- 主队列是串行队列，任务按顺序执行。
- 关键 API：dispatch_sync、dispatch_async、dispatch_get_main_queue。

<details><summary>原文</summary>

4.5 同步执行 + 主队列
同步执行 + 主队列 在不同线程中调用结果也是不一样，在主线程中调用会发生死锁问题，而在其他线程中调用则不会。

4.5.1 在主线程中调用 「同步执行 + 主队列」
互相等待卡住不可行
/**
* 同步执行 + 主队列
* 特点(主线程调用)：互等卡主不执行。
* 特点(其他线程调用)：不会开启新线程，执行完一个任务，再执行下一个任务。
*/
- (void)syncMain {
NSLog(@"currentThread---%@",[NSThread currentThread]); // 打印当前线程
NSLog(@"syncMain---begin");
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_sync(queue, ^{
// 追加任务 1
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"1---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_sync(queue, ^{
// 追加任务 2
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"2---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_sync(queue, ^{
// 追加任务 3
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"3---%@",[NSThread currentThread]); // 打印当前线程
});
NSLog(@"syncMain---end");
}
输出结果
2019-08-08 14:43:58.062376+0800 YSC-GCD-demo[17371:4213562] currentThread---<NSThread: 0x6000026e2940>{number = 1, name = main}
2019-08-08 14:43:58.062518+0800 YSC-GCD-demo[17371:4213562] syncMain---begin
(lldb)

在主线程中使用 同步执行 + 主队列 可以惊奇的发现：

追加到主线程的任务 1、任务 2、任务 3 都不再执行了，而且 syncMain---end 也没有打印，在 XCode 9 及以上版本上还会直接报崩溃。这是为什么呢？
这是因为我们在主线程中执行 syncMain 方法，相当于把 syncMain 任务放到了主线程的队列中。而 同步执行 会等待当前队列中的任务执行完毕，才会接着执行。那么当我们把 任务 1 追加到主队列中，任务 1 就在等待主线程处理完 syncMain 任务。而syncMain 任务需要等待 任务 1 执行完毕，才能接着执行。

那么，现在的情况就是 syncMain 任务和 任务 1 都在等对方执行完毕。这样大家互相等待，所以就卡住了，所以我们的任务执行不了，而且 syncMain---end 也没有打印。

要是如果不在主线程中调用，而在其他线程中调用会如何呢？

4.5.2 在其他线程中调用「同步执行 + 主队列」
不会开启新线程，执行完一个任务，再执行下一个任务
// 使用 NSThread 的 detachNewThreadSelector 方法会创建线程，并自动启动线程执行 selector 任务
[NSThread detachNewThreadSelector:@selector(syncMain) toTarget:self withObject:nil];
输出结果：
2019-08-08 14:51:38.137978+0800 YSC-GCD-demo[17482:4237818] currentThread---<NSThread: 0x600001dd6c00>{number = 3, name = (null)}
2019-08-08 14:51:38.138159+0800 YSC-GCD-demo[17482:4237818] syncMain---begin
2019-08-08 14:51:40.149065+0800 YSC-GCD-demo[17482:4237594] 1---<NSThread: 0x600001d8d380>{number = 1, name = main}
2019-08-08 14:51:42.151104+0800 YSC-GCD-demo[17482:4237594] 2---<NSThread: 0x600001d8d380>{number = 1, name = main}
2019-08-08 14:51:44.152583+0800 YSC-GCD-demo[17482:4237594] 3---<NSThread: 0x600001d8d380>{number = 1, name = main}
2019-08-08 14:51:44.152767+0800 YSC-GCD-demo[17482:4237818] syncMain---end

在其他线程中使用 同步执行 + 主队列 可看到：

所有任务都是在主线程（非当前线程）中执行的，没有开启新的线程（所有放在主队列中的任务，都会放到主线程中执行）。
所有任务都在打印的 syncConcurrent---begin 和 syncConcurrent---end 之间执行（同步任务 需要等待队列的任务执行结束）。
任务是按顺序执行的（主队列是 串行队列，每次只有一个任务被执行，任务一个接一个按顺序执行）。
为什么现在就不会卡住了呢？

因为syncMain 任务 放到了其他线程里，而 任务 1、任务 2、任务3 都在追加到主队列中，这三个任务都会在主线程中执行。syncMain 任务 在其他线程中执行到追加 任务 1 到主队列中，因为主队列现在没有正在执行的任务，所以，会直接执行主队列的 任务1，等 任务1 执行完毕，再接着执行 任务 2、任务 3。所以这里不会卡住线程，也就不会造成死锁问题。

4.6 异步执行 + 主队列
只在主线程中执行任务，执行完一个任务，再执行下一个任务。
/**
* 异步执行 + 主队列
* 特点：只在主线程中执行任务，执行完一个任务，再执行下一个任务
*/
- (void)asyncMain {
NSLog(@"currentThread---%@",[NSThread currentThread]); // 打印当前线程
NSLog(@"asyncMain---begin");
dispatch_queue_t queue = dispatch_get_main_queue();
dispatch_async(queue, ^{
// 追加任务 1
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"1---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_async(queue, ^{
// 追加任务 2
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"2---%@",[NSThread currentThread]); // 打印当前线程
});
dispatch_async(queue, ^{
// 追加任务 3
[NSThread sleepForTimeInterval:2]; // 模拟耗时操作
NSLog(@"3---%@",[NSThread currentThread]); // 打印当前线程
});
NSLog(@"asyncMain---end");
}
输出结果：
2019-08-08 14:53:27.023091+0800 YSC-GCD-demo[17521:4243690] currentThread---<NSThread: 0x6000022a1380>{number = 1, name = main}
2019-08-08 14:53:27.023247+0800 YSC-GCD-demo[17521:4243690] asyncMain---begin
2019-08-08 14:53:27.023399+0800 YSC-GCD-demo[17521:4243690] asyncMain---end
2019-08-08 14:53:29.035565+0800 YSC-GCD-demo[17521:4243690] 1---<NSThread: 0x6000022a1380>{number = 1, name = main}
2019-08-08 14:53:31.036565+0800 YSC-GCD-demo[17521:4243690] 2---<NSThread: 0x6000022a1380>{number = 1, name = main}
2019-08-08 14:53:33.037092+0800 YSC-GCD-demo[17521:4243690] 3---<NSThread: 0x6000022a1380>{number = 1, name = main}

在 异步执行 + 主队列 可以看到：

所有任务都是在当前线程（主线程）中执行的，并没有开启新的线程（虽然 异步执行 具备开启线程的能力，但因为是主队列，所以所有任务都在主线程中）。
所有任务是在打印的 syncConcurrent---begin 和 syncConcurrent---end 之后才开始执行的（异步执行不会做任何等待，可以继续执行任务）。
任务是按顺序执行的（因为主队列是 串行队列，每次只有一个任务被执行，任务一个接一个按顺序执行）。
弄懂了难理解、绕来绕去的「不同队列」+「不同任务」使用区别之后，我们来学习一个简单的东西：5. GCD 线程间的通信。

</details>

---
