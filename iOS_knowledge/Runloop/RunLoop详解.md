RunLoop详解

# RunLoop循环

RunLoop主要处理以下6类事件

* static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVE_CALLBACK_FUNCTION_()
* static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_BLOCK_()
* static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE_()
* static void __CFRUNLOOP_IS_CALLING_OUT_TO_TIME_CALLBACK_FUNCTION_()
* static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION_()
* static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION_()

1. Observer事件，runloop中状态变化时进行通知。（微信卡顿监控就是利用这个事件通知来记录下最近一次main runloop活动时间，在另一个check线程中用定时器检测当前时间距离最后一次活动时间过久来判断在主线程中的处理逻辑耗时和卡主线程）。这里还需要特别注意，CAAnimation是由RunloopObserver触发回调来重绘，接下来会讲到。
2. Block事件，非延迟的NSObject PerformSelector立即调用，dispatch_after立即调用，block回调。
3. Main_Dispatch_Queue事件：GCD中dispatch到main queue的block会被dispatch到main loop执行。
4. Timer事件：延迟的NSObject PerformSelector，延迟的dispatch_after，timer事件。
5. Source0事件：处理如UIEvent，CFSocket这类事件。需要手动触发。触摸事件其实是Source1接收系统事件后在回调 __IOHIDEventSystemClientQueueCallback() 内触发的 Source0，Source0 再触发的 _UIApplicationHandleEventQueue()。source0一定是要唤醒runloop及时响应并执行的，如果runloop此时在休眠等待系统的 mach_msg事件，那么就会通过source1来唤醒runloop执行。
6. Source1事件：处理系统内核的mach_msg事件。（推测CADisplayLink也是这里触发）

 [iOS 事件处理机制与图像渲染过程](http://www.cnblogs.com/yulang314/p/5091894.html)

