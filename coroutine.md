# Wrapping in a try-catch block
 * launch 에서는 일반 개발과 유사
```kotlin
scope.launch(Dispatchers.Default) {
    try {
        loggingService.upload(logs)
    } catch (e: Exception) {
        //Handle Exception
    }
}
```

 * async에서는 await에서 걸어줘야 한다!
```kotlin
suspend function getUser(userId: String): User {
    supervisorScope {
        val deferred = async(Dispatchers.IO) {
            userService.getUser(userId)
        }
        try {
            deferred.await()
        } catch (e: Exception) {
            //Handle exception
        }
    }
}
```

# Cancellation requires co-operation
 * 파일을 읽는 중에 cancel이 발생한다면? cooperate가 필요하다. 
```kotlin
scope.launch(Dispatchers.IO) {
    for(name in files) {
        readFile(name)
    }
}
```
# Check if the coroutine is Active
 * 해당 coroutine이 active상태인지 여부 확인
```kotlin
scope.launch(Dispatchers.IO) {
    for(name in files) {
        yield()//ensureActive() 
        readFile(name)
    }
}
```
 * 해당 coroutine이 cancel 가능할때마다 yield()를 호출하므로, coroutine의 실행을 stop 시킬 수 있다.
 * IO와 heavy computation일 경우 cancel가능여부를 체크할 수 있어야한다!

# When to mark a function as suspend
 * 언제 suspend라고 마킹해야하나?
 * 다른 suspend function을 호출할때마다!

```kotlin
suspend fun loadData() {
    val data = networkRequest()
    show(data)
}
```
 * loadData() 함수는 networkRequest()를 호출하므로 suspend를 붙여야 한다.
 * 왜 networkRequest()는 suspend 함수일까?
   * context를 가지고 호출되므로... context가 있으면 suspend 함수다.
   * coroutine library에서 그렇게 했으니까...

# When Not to mark a function as suspend
 * 다른 suspend 함수를 호출하지 않는 경우 
```kotlin
fun onButtonClicked() {
    scope.launch {
        loadData()
    }
}
```
 * onButtonClicked의 경우 그냥 함수이다.
 * launch로 coroutine을 trigger하는 경우, launch는 suspend 함수가 아니다!

# Unit testing coroutines
 * Use case 1
   * 해당 test는 새로운 coroutines를 trigger하지 않는 경우
   * loadData를 테스트하는 경우
     * async나 launch를 호출하지 않고 있다.
     * 그냥 suspend 함수이고 coroutine을 trigger하지 않고 있다.
     * context를 가지고 다른 thread로 이동하는게 networkRequest가 하는 일이다.
```kotlin
suspend fun loadData() {
    val data = networkRequest()
    show(data)
}
```
     * runBlocking은 coroutine library에 있는 mehtod로서 coroutine을 구동시킨다. 모든게 완료될때까지 thread를 block시킨다. loadData()이후에 순차적으로 실행된다.  
```kotlin
@Test
fun `Test loadData happy path`() = runBlocking {
    val viewModel = MyViewModel()
    viewModel.loadData()

    //뭔가 화면에 보여지는가?
}
```

## Use Case 2
 * 새로운 coroutine을 trigger하는 경우
```kotlin
class MyViewModel {
    val scope = CoroutineScope(
        Dispatchers.Main + SupervisorJob()
    )

    fun onButtonClicked() {
        scope.launch {
            loadData()
        }
    }
}
```
```kotlin
@Test
fun `Test loadData happy path`() = runBlocking {
    val viewModel = MyViewModel()
    viewModel.onButtonClicked()

    //뭔가 화면에 보여지는가?
}
```
 * onButtonClicked()을 테스트 하는 경우
   * launch를 사용하여 새로운 coroutine을 trigger하고 있다.
   * runBlocking과 같은 방식으로 사용 못한다.
     * loadData()가 다른 thread에서 실행되어 비동기로 실행되므로... 화면에 언제 보여질지 예측못한다.
   * 다른 방식 : 완료될때까지 기다리기
     * CoutDownLatch
     * LiveDataTestUtil
     * Mockito await
```kotlin
@Test
fun `Test loadData happy path`() = runBlocking {
    val viewModel = MyViewModel()
    viewModel.onButtonClicked()

    // Wait for the result -> using a CoutDownLatch or LiveDataTestUtil
    //뭔가 화면에 보여지는가?
}
```
     * 문제점 : 빠르게 실행할 수 없다. -> test에 적합하지 않다!


```kotlin
class MyViewModel {
    val scope = CoroutineScope(
        Dispatchers.Main + SupervisorJob()
    )

    fun onButtonClicked() {
        scope.launch {
            loadData()
        }
    }
}
```

```kotlin
class MyViewModel(
    // 인자로 disaptcher를 받을 수 있게하고 이 dispatcher로 trigger시키기
    private val dispatcher: CoroutineDispatcher
) {
    val scope = CoroutineScope(
        Dispatchers.Main + SupervisorJob()
    )

    fun onButtonClicked() {
        scope.launch(dispatcher) {
            loadData()
        }
    }
}
```

```kotlin
val testDispatcher = TestCoroutineDispatcher()

@Test
fun `Test buttonClicked happy path`() = testDispatcher.runBlockingTest {
    val viewModel = MyViewModel(testDispatcher)
    viewmodel.onButtonClicked()

    // 뭔가 보이는지 확인
}
```


```kotlin
class MyViewModel(
    private val dispatcher: CoroutineDispatcher
) {
    val scope = CoroutineScope(
        Dispatchers.Main + SupervisorJob()
    )

    fun onButtonClicked() {
        // Do something else coroutine실행하기 전에 처리할 일이 있는 경우 
        scope.launch(dispatcher) {
            loadData()
        }
    }
}
```

```kotlin
val testDispatcher = TestCoroutineDispatcher()

@Test
fun `Test buttonClicked happy path`() = testDispatcher.runBlockingTest {
    val viewModel = MyViewModel(testDispatcher)

    testDispatcher.pauseDispatcher()
    viewModel.onButtonClicked()
    // Assert onButtonClicked did something else. onButtonClicked()에서 lauch되기 전까지 부분에 대한 실행 확인 가능

    testDispatcher.resumeDispatcher()
    // Assert show did something. lauch실행 이후 테스트 가능
}
```
