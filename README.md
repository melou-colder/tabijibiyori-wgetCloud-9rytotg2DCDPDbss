
## 一：背景


### 1\. 讲故事


写这篇文章起源于训练营里一位朋友最近在微信聊到他对这个问题使用了一种非常切实可行，简单粗暴的方式，并且也成功解决了公司里几个这样的卡死dump，如今在公司已是灵魂级人物，让我也尝到了什么叫反哺！对，这个东西叫 `Harmony`, github网址： `https://github.com/pardeike/Harmony`，一个非常牛逼的C\#程序函数修改器。


## 二：卡死问题的回顾


### 1\. 故障成因


为了方便讲述，先把 WinForm/WPF 程序故障的调用堆栈给大家呈现一下。



```

0:000:x86> !clrstack
OS Thread Id: 0x4eb688 (0)
Child SP       IP Call Site
002fed38 0000002b [HelperMethodFrame_1OBJ: 002fed38] System.Threading.WaitHandle.WaitOneNative(System.Runtime.InteropServices.SafeHandle, UInt32, Boolean, Boolean)
002fee1c 5cddad21 System.Threading.WaitHandle.InternalWaitOne(System.Runtime.InteropServices.SafeHandle, Int64, Boolean, Boolean)
002fee34 5cddace8 System.Threading.WaitHandle.WaitOne(Int32, Boolean)
002fee48 538d876c System.Windows.Forms.Control.WaitForWaitHandle(System.Threading.WaitHandle)
002fee88 53c5214a System.Windows.Forms.Control.MarshaledInvoke(System.Windows.Forms.Control, System.Delegate, System.Object[], Boolean)
002fee8c 538dab4b [InlinedCallFrame: 002fee8c] 
002fef14 538dab4b System.Windows.Forms.Control.Invoke(System.Delegate, System.Object[])
002fef48 53b03bc6 System.Windows.Forms.WindowsFormsSynchronizationContext.Send(System.Threading.SendOrPostCallback, System.Object)
002fef60 5c774708 Microsoft.Win32.SystemEvents+SystemEventInvokeInfo.Invoke(Boolean, System.Object[])
002fef94 5c6616ec Microsoft.Win32.SystemEvents.RaiseEvent(Boolean, System.Object, System.Object[])
002fefe8 5c660cd4 Microsoft.Win32.SystemEvents.OnUserPreferenceChanged(Int32, IntPtr, IntPtr)
002ff008 5c882c98 Microsoft.Win32.SystemEvents.WindowProc(IntPtr, Int32, IntPtr, IntPtr)
...


```

这个程序之所以被卡死，底层原因到底大概是这样的。


1. 程序在t1时间，有非主线程创建了控件。
2. 程序在t2时间，用户主动或被动做了 远程连接，Windows主题色刷新 等操作，这种系统级操作Windows需要同步刷新给所有UI控件。
3. 那些非主线程控件由于没有 MessageLoop 机制，导致主线程给这些UI发消息时得不到响应，最终引发悲剧。


t2时间的卡死是由于t1时间的错误创建导致，要想在dump中反向追溯目前是无法做到的，所以要想找到祸根需要监控t1，即`MarshalingControl`到底是谁创建的，为此我也写过两篇文章来仔细分析此事。


* (一个超经典 WinForm 卡死问题的再反思 )\[[https://github.com/huangxincheng/p/16868486\.html](https://github.com)]
* (一个超经典 WinForm 卡死问题的最后一次反思)\[[https://github.com/huangxincheng/p/17654394\.html](https://github.com):[Flowercloud 机场订阅加速](https://flowercloud6.com)]


第一种方式是启动 windbg 对 `System_Windows_Forms_ni System.Windows.Forms.Application+MarshalingControl..ctor` 进行拦截，说实话这种方式很多程序员搞不定，原因在于windbg的使用门槛较高，现实中很多程序员连CURD都没摸明白，所以可想而知了。。。


第二种方式是启动 perfview 对 winform/wpf 程序进行监控，直到程序出现卡死停止收集。最后在录播中寻找 `MarshalingControl..ctor` 的调用栈，这种方式也有不可行的时候，如果说卡死发生在程序启动的10天后，那这个录播文件将会超级超级大，或者有更极端的情况发生。


所以这两种方案都有各自的优缺点，现实可行性虽然有，但不高。。。今天作为终结篇，必须把这个问题安排掉，继续提供两种切实可行的方案。


## 三：两种修改方案


### 1\. 使用 Harmony 注入


Harmony作为一款运行时C\#方法修改器，借助它我完全可以将一些逻辑注入到 `MarshalingControl..ctor` 中，比如记录下初始化该方法的 `堆栈信息` ，是不是就可以轻松找到这个非主线程控件到底是谁？对不对，有了思路，我们在 nuget 上引用 `Lib.Harmony` ，上代码说话。



```

    public partial class Form1 : Form
    {
        public Form1()
        {
            InitializeComponent();

            var harmony = new Harmony("一线码农聊技术");

            Type applicationType = typeof(Application);
            Type marshalingControlType = applicationType.GetNestedType("MarshalingControl", BindingFlags.NonPublic);
            ConstructorInfo constructor = marshalingControlType.GetConstructor(BindingFlags.NonPublic | BindingFlags.Instance, null, Type.EmptyTypes, null);

            var prefix = typeof(HookMarshalingControl).GetMethod("OnActionExecuting");

            harmony.Patch(constructor, new HarmonyMethod(prefix));
        }

        private void Form1_Load(object sender, EventArgs e)
        {
        }

        private void backgroundWorker1_DoWork(object sender, DoWorkEventArgs e)
        {
            Button btn = new Button();
            var query = btn.Handle;
        }

        private void button1_Click(object sender, EventArgs e)
        {
            backgroundWorker1.RunWorkerAsync();
        }
    }

    /// 
    /// Hook MarshalingControl 的描述类
    /// 
    public class HookMarshalingControl
    {
        /// 
        /// 原生方法之前执行的 action
        /// 
        public static void OnActionExecuting()
        {
            Console.WriteLine("----------------------------");
            Console.WriteLine($"控件创建线程:{Thread.CurrentThread.ManagedThreadId}");
            Console.WriteLine(Environment.StackTrace);
            Console.WriteLine("----------------------------");
        }
    }


```

卦中的代码逻辑我就不详述了，核心就是将 `OnActionExecuting` 方法注入到 `MarshalingControl..ctor` 构造函数里，把程序运行起来后观察 output 窗口，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250113121037568-835443704.png)


终于是一个卧槽，祸根居然是一个 `tid=3` 的线程初始化了 `new Button()` 控件。。。


### 2\. 使用 DnSpy


Harmony 作为一款修改器，它对程序的侵入性是非常高的，目前还是有一些bug，比如对 .NET7 的支持还不是很好，但相对 `perfview` 和 `windbg` 的方式已经非常轻量级了，极大的降低了使用门槛。


问题来了，那有没有一种对程序无侵入，可行性超高的方式呢？当然是有的，dnspy 此时可以闪亮登场，用过 dnspy 的朋友应该知道它是一款轻量级，免安装绿色的调试器，当然除了调试器功能，它还是一款程序集修改器，可以实现 Harmony 的所有功能，在实践中我们可以将 dnspy copy 到客户机使用 `启动调试` 或者 `附加进程` 的方式对程序进行干预。


如何使用 dnspy 对 `MarshalingControl..ctor` 进行干预呢？可以使用 `断点日志` 的功能，日志信息如下：



> `控件创建线程：{Environment.CurrentManagedThreadId} \n $CALLSTACK`


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250113121037559-652867249.png)


有些人可能要问了 `$CALLSTACK` 是什么东西？很显然是堆栈信息，除了这个关键词还有很多，具体可以看后面的 `问号面板`。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250113121037564-2009035290.png)


接下来把程序跑起来，观察 output面板。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250113121037569-1092479790.png)


从面板中可以清楚的看到，原来有个 tid\=3 的线程创建了一个 `Button` 控件，这就是我们要找的祸根。


到这里，可能有些人要说，dnspy 启动 exe 的方式因为各种原因在我们这边行不通，有没有其他的方式呢？ 当然是有的，我们还可以在程序启动之后以 `进程附加` 的方式注入，同样也是一种非常可行且低侵入的方式。


为了能够更早的介入，可以在 Form1 初始化之前弹一个MessageBox，有更好的方式大家也可以说一下，感谢。参考代码如下：



```

    public partial class Form1 : Form
    {
        public Form1()
        {
            MessageBox.Show("开启你的注入吧...");
            InitializeComponent();
        }
    }


```

弹框之后，使用 dnspy 的进程附加。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250113121037578-965890344.png)


附加好了之后关闭弹框让程序继续运行，点击 buttton 按钮，可以看到 output 上的输出。



```

11:20:01.548 控件创建线程：<<<当线程位于不安全状态时无法对表达式进行求值。按步调试或运行直到触发断点。>>>  
11:20:01.550   	System.Windows.Forms.Application.MarshalingControl.MarshalingControl
11:20:01.551 	System.Windows.Forms.Application.ThreadContext.MarshalingControl.get
11:20:01.552 	System.Windows.Forms.WindowsFormsSynchronizationContext.WindowsFormsSynchronizationContext
11:20:01.553 	System.Windows.Forms.WindowsFormsSynchronizationContext.InstallIfNeeded
11:20:01.553 	System.Windows.Forms.Control.Control
11:20:01.554 	System.Windows.Forms.ButtonBase.ButtonBase
11:20:01.554 	System.Windows.Forms.Button.Button
11:20:01.554 	WindowsFormsApp1.Form1.backgroundWorker1_DoWork
11:20:01.555 	System.ComponentModel.BackgroundWorker.OnDoWork
11:20:01.555 	System.ComponentModel.BackgroundWorker.WorkerThreadStart
11:20:01.556 	System.Runtime.Remoting.Messaging.StackBuilderSink.AsyncProcessMessage
11:20:01.556 	System.Threading.ExecutionContext.RunInternal
11:20:01.557 	System.Threading.ExecutionContext.Run
11:20:01.557 	System.Threading.QueueUserWorkItemCallback.System.Threading.IThreadPoolWorkItem.ExecuteWorkItem
11:20:01.557 	System.Threading.ThreadPoolWorkQueue.Dispatch
11:20:01.558 	[本机到托管的转换]
11:20:01.558 	  


```

这里稍微提醒一下，tid 在这里没有显示出来，大家可以换成`问号面板`上的关键词 `$TID` 即可，不过TID不是最重要的，最重要的是调用栈给弄出来了。


## 四：总结


作为一名专业的 `.NET高级调试师`，在这个经典卡死的问题溯源上一直没有提供非常好的解决方案，还是有些内疚的，在我的高级调试之旅中还是会不间断的收到类似dump，相信这篇文章之后，不再有人被它所困扰！
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


