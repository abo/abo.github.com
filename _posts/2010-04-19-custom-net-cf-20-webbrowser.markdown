---
layout: post
title: .NET CF 2.0 WebBrowser控件定制
---

.net compact framework 2.0提供的webbrowser control有两个很明显的标志：

1. 在加载内容时会在控件底部显示一个状态栏；

1. ContextMenu中有“收藏夹”、“刷新”之类的东西。

.net cf没有提供相关的API，不过好在有P/Invoke。

1. 隐藏状态栏

用Remote Spy连接到模拟器上，可以看到WebBrowser由两个Window组成：MSPIE Status和PIEHTML，其中MSPIE Status就是状态栏窗口，想办法隐藏它就可以了。以下是示例代码：

Dll导入声明及工具方法：

{% highlight csharp %}
		#region DllImports
    /// <summary>
    /// 状态栏的窗口名
    /// </summary>
    private const string StatusBarClassName = "MSPIE Status";

    [DllImport("coredll.dll")]
    public static extern int GetClassName(IntPtr hWnd, StringBuilder lpClassName, int nMaxCount);

    [DllImport("coredll.dll")]
    public static extern IntPtr GetWindow(IntPtr hwnd, int cmd);

    [DllImport("coredll.dll")]
    public static extern bool DestroyWindow(IntPtr hwnd);

    public enum GetWindowFlags : int
    {
        GW_HWNDFIRST = 0,
        GW_HWNDLAST = 1,
        GW_HWNDNEXT = 2,
        GW_HWNDPREV = 3,
        GW_OWNER = 4,
        GW_CHILD = 5,
        GW_MAX = 5
    }

    public static IntPtr FindHwnd(string className, IntPtr parentHwnd)
    {
        StringBuilder sbClass = new StringBuilder(256);
        if (0 != GetClassName(parentHwnd, sbClass, sbClass.Capacity) && sbClass.ToString() == className)
        {
            return parentHwnd;
        }

        IntPtr hwndChild = GetWindow(parentHwnd, (int)GetWindowFlags.GW_CHILD);
        while (hwndChild != IntPtr.Zero)
        {
            IntPtr result = FindHwnd(className, hwndChild);
            if (result != IntPtr.Zero)
                return result;

            hwndChild = GetWindow(hwndChild, (int)GetWindowFlags.GW_HWNDNEXT);
        }

        return IntPtr.Zero;
    }
    #endregion
{% endhighlight %}
    
在父控件的构造函数里去掉状态栏窗口：

{% highlight csharp %}
		IntPtr hwndStatus = FindHwnd(StatusBarClassName, body.Handle);
    if (hwndStatus != IntPtr.Zero)
    {                       
        DestroyWindow(hwndStatus);                
    }
{% endhighlight %}

这时会发现状态栏窗口是没了，但是就在WebBrowser控件的底部，原来状态栏窗口的位置，出现了一个矩形空白区域。调整一下WebBrowser控件的高度就行了：

{% highlight csharp %}
		/// <summary>
    /// 状态栏窗口的高度
    /// </summary>
    private int StatusBarHeight;  
    
    [DllImport("coredll.dll")]
    static extern bool GetClientRect(IntPtr hWnd, out RECT lpRect);
    
		[StructLayout(LayoutKind.Sequential)]
    public struct RECT
    {
        public int X;
        public int Y;
        public int Width;
        public int Height;
    }    
{% endhighlight %}

在去掉状态栏窗口前计算一下高度，保存起来：
        
{% highlight csharp %}        
    RECT rectStatus = new RECT();              
    GetClientRect(hwndStatus, out rectStatus);
    StatusBarHeight = rectStatus.Height;
    DestroyWindow(hwndStatus);     
{% endhighlight %}    

然后在父控件的大小变化时调整WebBrowser的高度：

{% highlight csharp %}
		browser.Height = this.Height + StatusBarHeight;		
{% endhighlight %} 

1. 禁用ContextMenu

将发给WebBrowser的ContextMenu相关消息截获，忽略掉就行了。这需要声明自己的消息处理函数，将原来的消息处理函数替换下来，然后在自己的消息处理函数里将WM_CONTEXTMENU以外的其他消息都转发到原消息处理函数，示例代码：

Dll导入声明及工具方法：

{% highlight csharp %}
    /// <summary>
    /// 浏览器控件的窗口名
    /// </summary>
    private const string IEHTMLClassName = "PIEHTML";
    public const int GWL_WNDPROC = (-4);

    [DllImport("coredll.dll")]
    public static extern int GetClassName(IntPtr hWnd, StringBuilder lpClassName, int nMaxCount);

    [DllImport("coredll.dll")]
    public static extern IntPtr GetWindow(IntPtr hwnd, int cmd);

    public delegate IntPtr WndProcDelegate(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam);

    [DllImport("coredll", SetLastError = true)]
    public static extern IntPtr GetWindowLong(IntPtr hWnd, int nIndex);

    [DllImport("coredll", SetLastError = true)]
    public static extern int SetWindowLong(IntPtr hWnd, int nIndex, IntPtr newWndProc);

    [DllImport("coredll", SetLastError = true)]
    public static extern IntPtr CallWindowProc(IntPtr lpPrevWndFunc, IntPtr hWnd, uint Msg, IntPtr wParam, IntPtr lParam);

    public enum GetWindowFlags : int
    {
        GW_HWNDFIRST = 0,
        GW_HWNDLAST = 1,
        GW_HWNDNEXT = 2,
        GW_HWNDPREV = 3,
        GW_OWNER = 4,
        GW_CHILD = 5,
        GW_MAX = 5
    }

    public static IntPtr FindHwnd(string className, IntPtr parentHwnd)
    {
        StringBuilder sbClass = new StringBuilder(256);
        if (0 != GetClassName(parentHwnd, sbClass, sbClass.Capacity) && sbClass.ToString() == className)
        {
            return parentHwnd;
        }

        IntPtr hwndChild = GetWindow(parentHwnd, (int)GetWindowFlags.GW_CHILD);
        while (hwndChild != IntPtr.Zero)
        {
            IntPtr result = FindHwnd(className, hwndChild);
            if (result != IntPtr.Zero)
                return result;

            hwndChild = GetWindow(hwndChild, (int)GetWindowFlags.GW_HWNDNEXT);
        }

        return IntPtr.Zero;
    }
{% endhighlight %}
        
自己的消息处理：

{% highlight csharp %}
	  /// <summary>
	  /// 默认的窗口消息处理函数
	  /// </summary>
	  private IntPtr originalWndProc = IntPtr.Zero;
	  /// <summary>
	  /// 新的窗口消息处理函数
	  /// </summary>
	  private WndProcDelegate customWndProc;
	
		public IntPtr IgnoreContextMenuWndProc(IntPtr hWnd, uint msg, IntPtr wParam, IntPtr lParam)
	  {
	      switch (msg)
	      {
	          case 0x007B://WM_CONTEXTMENU
	              return IntPtr.Zero;
	          default:
	              break;
	      }
	      return CallWindowProc(originalWndProc, hWnd, msg, wParam, lParam);
	  }
  {% endhighlight %}
  
在父控件的构造函数里将WebBrowser的消息处理函数换掉：

{% highlight csharp %}
		IntPtr hwndIEHtml = FindHwnd(IEHTMLClassName, body.Handle);
		if (hwndIEHtml != IntPtr.Zero)
		{
		    customWndProc = new WndProcDelegate(IgnoreContextMenuWndProc);
		    originalWndProc = GetWindowLong(hwndIEHtml, GWL_WNDPROC);
		    SetWindowLong(hwndIEHtml, GWL_WNDPROC, Marshal.GetFunctionPointerForDelegate(customWndProc));
		}
{% endhighlight %}