# 什么时候会将数据上报？

1. 检测到（`/data/anr/` 目录）ANR 日志有新的记录（检测频率：至少间隔30分钟，默认2小时）；
2. 网络变化后，当前处于 WIFI 连接，且距上次上报间隔超过配置时间间隔，且有未上传的数据（检测频率：至少5分钟，默认1小时）。


# 上报数据的通用格式？

*相关源码：`UploadManager.upload`。*

`IUpload.upload` 接口收到的数据类型为 `Map<String, String>`，其中包含一个键值对，key 为 `apm_data` 的键值存储了 `JSON` 格式的数据。


## 该 `JSON` 一定包含的基础属性字段有：

- **m**:
`Build.MODEL`，手机型号;

- **mf**:
`Build.MANUFACTURER`，手机厂商;

- **s**:
`Build.VERSION.SDK_INT`，系统版本号;

- **rv**:
`Build.VERSION.RELEASE`，系统版本名;

- **fv**:
`BuildConfig.APPLICATION_ID`，应用 id;

- **apmid**:
`APM id`，应用 APM id;

- **apmver**:
`APM SDK version`，APM SDK 的版本;

- **t**:
`System.currentTimeMillis()`，系统当前时间。

该 `JSON` 还包含其他的字段，分别对应各项监控数据。单个监控数据如同一张表格，如：`activity` 的值对应一个数组（JSONArray），而数组的每一项为一条记录对象（JSONObject），所有记录对象都包含一个 `tr` 字段，对应了记录的产生时间。


## 不同监控数据如下：

- **activity**:

*相关源码：`ActivityInfo`, `ActivityCore`。*

记录 Activity 的生命周期函数的执行（超过 100ms）。**首帧** 为 `activity.getWindow().getDecorView().post()` 获取回调的时间。

具体包含如下内容：

    + tr: 记录时间
    + n: Activity 名称
    + st: 启动类型，1: 冷启动，0: 热启动
    + t: 生命周期函数执行耗时，单位（毫秒）
    + lc: 生命周期，包括：onCreate/onStart/onResume/onPause/onStop/onDestroy/首帧(距离调用 onCreate 超过 300ms)
    + an: 页面对应的插件名称（用在插件化环境中）
    + av: 页面对应的插件版本（用在插件化环境中）

- **anr**:

*相关源码：`AnrInfo`, `AnrFileParser`。*

记录应用发生的 ANR 事件。

    + P: 进程名
    + c: ANR 事件详细内容
    + pid: 进程id
    + t: ANR 发生时间

- **appstart**:

*相关源码：`AppStartInfo`, `ActivityCore`。*

应用启动耗时，对应应用的页面 **第一帧** 距离开始调用 `Application.onCreate` 的间隔时间。**第一帧** 为第一次 `activity.getWindow().getDecorView().post()` 获取回调的时间。

    + tr: 记录时间
    + st: 启动耗时，单位（毫秒）

- **block**:

*相关源码：`BlockInfo`, `BlockTask`。*

通过 `Looper.getMainLooper().setMessageLogging` 检测主线程单个 message dispatch 是否超过 4.5s，记录超时发生时的主线程调用栈。

    + tr: 记录时间
    + pn: 进程名
    + stack: 主线程调用栈
    + bt: 当前的超时检测阈值，单位（毫秒）

- **fileinfo**:

*相关源码：`FileInfo`, `FileInfoTask`。*

检测 `/sdcard/xyz`、`/app/files/xyz` 以及 `/app/files/databases` 目录下大小超过 50k 的文件或目录。

    + tr: 记录时间
    + fn: 文件名称
    + fp: 文件完整路径
    + ft: 文件类型，目录/文件
    + lm: 最后修改时间
    + fs: 文件大小，单位（字节数）
    + fw: 是否可写
    + fr: 是否可读
    + fe: 是否可执行
    + sfn: 包含的子文件个数

- **fps**:

*相关源码：`FpsInfo`, `FpsTask`。*

通过 `Choreographer.postFrameCallback` 检测 fps 数，记录帧数低于 30 的场景。

    + tr: 记录时间
    + f: fps 数
    + ac: Activity 名称
    + ty: 检测方式，固定为0：CHOREOGRAPHER
    + pn: 进程名
    + par: 当前（后台）线程调用栈（个人觉得没用处）

- **func**:

*相关源码：`FuncInfo`, `FuncTrace`。*

通过 AOP 记录所有 `run()` 以及 `onReceive(Context, Intent)` 函数的执行耗时，对于 `run()` 执行超过 4s，`onReceive(Context, Intent)` 超过 2s 的场景进行上报。

    + tr: 记录时间
    + ty: 函数类型，run/onReceive
    + par: 包含 {进程名,线程名,线程id,执行耗时,源码位置,调用堆栈,action参数} 这些详细信息

- **memory**:

*相关源码：`MemoryInfo`, `MemoryTask`。*

通过 Debug.getMemoryInfo() 获得内存使用信息，从应用启动 10s 后开始，每隔 30 分钟收集一次，进行上报。

    + tr: 记录时间
    + pn: 进程名
    + tp: info.getTotalPss()
    + dp: info.dalvikPss
    + np: info.nativePss
    + op: info.otherPss

- **net**:

*相关源码：`NetInfo`, `QOKHttp`, `NetWorkInterceptor`。*

通过 AOP 或 Interceptor 统计 `URL.openConnection`、 `apache-http-client` 以及 `okhttp3` 的请求耗时信息，进行上报。

    + tr: 记录时间
    + u: 请求链接（移除了 query 部分）
    + sc: 响应状态码
    + ec: 响应码（未使用，固定为0）
    + sb: 发送的字节数（并不精确，统计了 url 和 body）
    + rb: 响应的字节数（并不精确，统计了 body）
    + w: 是否 wifi 连接
    + t: 请求开始时间
    + tc: 从请求开始到响应完成的耗时

- **processinfo**:

*相关源码：`ProcessInfo`, `ProcessInfoTask`。*

统计进程启动次数，进程启动 2s 后开始记录。

    + tr: 记录时间
    + pn: 进程名
    + sc: 启动次数（固定为1）

- **watchdog**:

*相关源码：`WatchDogInfo`, `WatchDogTask`。*

通过 `new Handler(Looper.getMainLooper()).post` 检测主线程批量 message dispatch 是否超过 4.5s，记录超时发生时的主线程调用栈。

统计的数据与 **block** 相同。

    + tr: 记录时间
    + pn: 进程名
    + stack: 主线程调用栈
    + bt: 当前的超时检测阈值，单位（毫秒）

- **webview**:

*相关源码：`WebInfo`, `WebTrace`, `JSBridge`。*

通过 AOP 在 `WebViewClient.onPageFinished` 中调用 js 获取 `window.performance.timing` 数据，进行上报。

    + tr: 记录时间
    + u: 页面链接
    + w: 是否 wifi 连接
    + ns: window.performance.timing.navigationStart
    + rs: window.performance.timing.responseStart
    + pt: 页面加载完成时的系统当前时间


# 通过云端规则控制行为

各项监控是否启用、收集的间隔时间、以及指标的阈值都可通过云端进行下发控制。

对应通过 `IRuleRequest.request` 获得云端的规则。


# 对应源码

本次分析的源码版本及链接为：https://github.com/Qihoo360/ArgusAPM/tree/75ead19ca98a8a1f776688e9df5b572f20c80b12
