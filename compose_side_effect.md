在 Compose 中，Composable 函数的核心原则是它们应该是无副作用 (side-effect-free) 的。这意味着它们应该只负责描述 UI，而不应该修改 App 的状态或执行任何可能与 Composable 函数生命周期无关的操作。
然而，在实际应用中，我们不可避免地需要与 UI 外部的世界进行交互，例如：
* 启动一个网络请求。
* 从数据库读取数据。
* 响应 back 按键。
* 当一个 Composable 进入屏幕时，发送一个分析事件。
这些操作就属于“副作用”。为了在 Composable 的生命周期内安全地处理它们，Compose 提供了一系列特定的 Effect API。
以下是 Compose 中主要的 Side Effect API：
## LaunchedEffect
这是最常用的 Effect 之一。它用于在一个 Composable 的生命周期内启动一个协程（coroutine），执行那些需要在后台安全执行的“一次性”挂起操作。
使用场景：当 Composable 首次进入组合（Composition）时，执行一个异步任务，比如网络请求、数据库操作、或者显示一个 Snackbar。
关键特性：
* 接收一个或多个 key 参数。当 key 的值发生变化时，LaunchedEffect 会取消正在运行的协程，并用新的 key 值启动一个新的协程。
* 如果 key 保持不变，协程将在 Composable 存在于组合中时一直运行，并在它离开组合时自动取消，从而避免内存泄漏。
```Kotlin
Scaffold(snackbarHost = { SnackbarHost(snackState) }) { innerPadding ->
    LaunchedEffect(error) {
        error?.let { snackState.showSnackbar(it) }
    }
    Box(
        Modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.primaryContainer)
            .padding(innerPadding)

    ) {
        Text(text = error.toString())
    }
    LaunchedEffect(Unit) {
        repeat(5) {
            error = Random.nextInt().toString()
            delay(2000L)
        }
    }
}
```
工作流程：
1. 外面的 LaunchedEffect(Unit) 每3秒改变一次 error 变量的值。
2. LaunchedEffect(error) 监听到 error 有了新值。
3. 它会取消上一个还在运行的协程（如果 showSnackbar 还没显示完），并用新的 error 值启动一个新协程。
4. 新协程调用 snackState.showSnackbar(it)，SnackbarHost 监听到状态变化，从而显示新的 Snackbar。
5. key 为 Unit 的 LaunchedEffect 表示这个协程只在 Composable 首次进入组合时启动一次，之后不再重启。

关键点：LaunchedEffect 将一个或多个变量 (key) 作为依赖。只有当 key 变化时，协程才会重启。这是在 Composable 的生命周期内执行异步任务最直接的方式。


## DisposableEffect 
```Kotlin
@Composable
private fun DisposableEffectTest() {
    Box(Modifier.safeContentPadding()) {
        var isConnected by remember { mutableStateOf(null as Boolean?) }
        val context = LocalContext.current
        DisposableEffect(context) {
            val connectivityManager =
                context.getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
            val networkCallback = object : ConnectivityManager.NetworkCallback() {
                override fun onAvailable(network: Network) {
                    super.onAvailable(network)
                    isConnected = true
                }

                override fun onLost(network: Network) {
                    super.onLost(network)
                    isConnected = false
                }
            }
            val networkRequest = NetworkRequest.Builder()
                .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
                .build()
            connectivityManager.registerNetworkCallback(networkRequest, networkCallback)
            onDispose {
                connectivityManager.unregisterNetworkCallback(networkCallback)
            }
        }
        Text("Network connected: $isConnected")
    }
}
```
工作流程
1. 何时执行“效果”？
* 当 DisposableEffectTest Composable 首次被渲染到屏幕上时，DisposableEffect 的主代码块会执行。在例子中，就是注册 networkCallback 的过程。
2. 何时执行“清理” (onDispose)？
* 当 DisposableEffectTest Composable 从屏幕上消失时（例如，关闭了应用或导航到了另一个界面），onDispose 块会自动执行，从而调用 unregisterNetworkCallback。
* （更精细的控制） 当 key (你用的是 context) 发生变化时，onDispose 会先被调用来清理旧的效果，然后主代码块会再次执行以创建新的效果。
关键要点总结
* 用途：专门用于处理需要手动释放或注销的资源。如果副作用操作是 addListener，那么就一定需要一个 removeListener 与之对应。这就是 DisposableEffect 的用武之地。
* 结构：它的 lambda 表达式必须返回一个 onDispose {} 块。这是它的标志性结构，也是它与 LaunchedEffect 的最大区别。
* 防止内存泄漏：它是保证 Android 组件（如 BroadcastReceiver, Callback, SensorEventListener 等）被正确注销、从而防止内存泄漏的标准且可靠的方式。
* Key 的重要性：key 决定了 Effect 的重启时机。
  * 使用 Unit 作 key：表示“只在进入时执行，只在最终离开时清理”。
  * 使用 context 作 key：表示“当 context 变化时，请帮我自动清理旧的监听并用新的 context 注册新的监听”，这是一种更健壮的写法。

当需要在 Composable 的生命周期内安全地使用那些需要成对的 register/unregister 或 subscribe/unsubscribe 方法的传统 Android API 时，DisposableEffect 是首选工具。

## rememberCoroutineScope - 在用户事件回调中启动协程
```Kotlin

@Composable
private fun RememberCoroutineScopeTest() {
    Box(Modifier.safeContentPadding()) {
        val scope = rememberCoroutineScope()
        val snackState = remember { SnackbarHostState() }
        Scaffold(snackbarHost = { SnackbarHost(snackState) }) {
            Button(
                onClick = {
                    scope.launch {
                        println("start time consuming task...")
                        delay(2000L)
                        println("end time consuming task...")
                        snackState.showSnackbar("Task completed")
                    }
                },
            ) {
                Text("Click")
            }
        }
    }
}
```
场景： LaunchedEffect 是自动触发的。但如果想在用户点击一个按钮后，才去执行一个耗时操作（比如上传文件或保存数据到数据库），应该怎么办？这时就需要 rememberCoroutineScope。
工作流程:
1. rememberCoroutineScope() 创建一个 CoroutineScope。这个 scope 会在 Composable 离开组合时自动取消所有它启动的子协程。
2. 当用户点击按钮时，onClick lambda 被触发。
3. 使用 scope.launch 在这个与 UI 生命周期绑定的作用域中安全地启动一个协程。
4. 即使用户在 delay(1500L) 完成前离开这个界面，该协程也会被自动取消，防止了不必要的工作和潜在的崩溃。
与 LaunchedEffect 的区别：
* LaunchedEffect: 自动在 key 变化时执行。
* rememberCoroutineScope: 提供一个 scope，在事件回调（如 onClick）中启动协程。
