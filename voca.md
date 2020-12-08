# 용어
 * context
 * job
 * dispatcher

# CoroutineScope
 * launch나 async로 생성한 coroutine을 추적
 * extension functions
 * 언제 생성?
   * app의 특정 layer에서 uncontrolled lifecycle을 구동하는 경우
 * 사용법
```kotlin
class ExampleClass {
    val scope = CoroutineScope() //scope 생성

    fun loadExample() {
        scope.launch { //이 scope으로 새로운 coroutine들을 생성할 수 있음
            // 새로운 coroutine을 launch
            // suspend function 호출할 수 있음!
        }
    }

    fun cleaup() {
        scope.cancel()
    }
}
```
 * coroutine할 때 launch를 추천하는 이유
   * async와 달리 결과를 return하지 않음
     * async는 다른 coroutine을 필요할 수 있고 이 경우 이 coroutine의 sub로 취급될 수 있다.

 * ExampleClass instance 생성될때마다 cleanup 해줘야 한다. 
   * 한꺼번에 cancel 및 stop 실행

# viewModelScope lifecycleScope
 * scopes을 수동으로 생성 및 관리하는 경우
   * 원래 boilerplate 코드가 필요해서 
   * 편리하게 사용할 수 있도록 android에서는 KTX library를 제공
 * 특정 lifecycle class 내부에서 coroutine scope을 제공
   * ViewModel class를 위해서 viewModelScope을 제공
   * lifecycle owner를 위해서 lifecycleScope를 제공
   * 안드로이드에서 해당 class에 필요한 것을 ktx library에서 제공해 주는 것과 같음
   * 아마 test를 위해서 그런게 아닐까?

```kotlin
class ExampleViewModel : ViewModel() {
    fun loadExample() {
        viewModelScope.launch { //ViewModel 상속 받은 class에서 viewModelScope 사용 가능
            // 새로운 coroutine
            // suspend function 호출 가능
        }
    }
}
```
 * viewModelscope에서 launch된 coroutine은 ViewModel이 clear될때 자동으로 cancel된다.

# lifecycleScope lifecycle
```kotlin
class ExampleActivity : AppCompatActivity() {

    fun loadAsset() {
        lifecycleScope.launch { //Activity 상속 받은 class에서 사용 가능
            // 새로운 coroutine
            // suspend function 호출 가능
        }
    }
}
```
 * viewmodelScope, lifecycleScope는 main thread에서 coroutine을 실행
 * 해당 scope은 다른 dispatcher를 원하는 경우 아래와 같이 Dispatchers.Domain을 설정
```kotlin
class ExampleActivity : AppCompatActivity() {
    fun loadAsset() {
        lifecycleScope.launch(Dispatchers.IO) { //Dispatchers.IO 도메인 설정. IO에 최적화된 thread rule
            // 새로운 coroutine
            // suspend function 호출 가능
        }
    }
}
```
 * launch는 CoroutineContext를 parameter를 가진다. CoroutineScope도 역시 가능.
```kotlin
fun CoroutineScope.launch {
    context: CoroutineContext = EmptyCoroutineContext, 
    ...
}

fun CoroutineScope(
    context: CoroutineContext
): CoroutineScope
```

# CoroutineContext
 * coroutine의 behavior를 정의하는 element의 index
 * 4가지 elements
   * CoroutineDispatcher
     * Dispatchers.IO
     * Dispatchers.Default
     * Dispatchers.Main
   * CoroutineExceptionHandler
   * CoroutineName
   * Job
     * coroutine과 coroutineScope의 lifecycle을 제어
```kotlin
class ExampleClass {
    val scope = CoroutineScope(
        Job()
    )
}
```
 * scope를 cancle 방법
   * scope.cancel 이용
   * Job을 이용
 * 여러 CoroutineContext를 결합하기 : '+' 연산자를 이용
```kotlin
class ExampleClass {
    val scope = CoroutineScope(
        Job() + Dispatchers.Main + exceptionHandler
    )

    fun loadExample() {
        // 새로운 Job instance 생성
        val job = scope.launch {
            // 상속받은 context
            // Dispatchers.Main을 사용
        }
    }
}
```

```kotlin
class ExampleClass {
    val scope = CoroutineScope(...)

    fun loadExample() {
        val job = scope.launch {
            // 여기서 Dispatchers.Main을 사용
            withContext(Dispatchers.Default) {
                // 여기서는 Dispatchers.Default를 사용
            }
        }
    }
}
```

# Job
 * coroutine의 핵심
 * 새로운 coroutine은 lifecyle을 제어할 수 있는 특정 job을 반환한다.

```kotlin
class ExampleViewModel : ViewModel() {

    fun loadExample() {
        val job = viewModelScope.launch { ... }
    }

    try {
        // business logic
    } catch(e: Throwable) { //exception이 걸리면 job을 취소하도록
        job.cancel()
    }
}
```
 * launch를 이용해서 새로운 coroutine을 생성하고 job이라는 변수에 반환값을 저장
 * job을 수행하는 동안 exception 상황이 발생하면 job을 취소 가능
```kotlin
class ExampleClass {
    val scope = CoroutineScope(Job())
}
```
 * job은 error가 어떻게 전파되는지에 영향을 미친다.
 * Job()으로 CoroutineScope을 생성하고 4개 Coroutine를 launch하는 상황

```bash
                Scope (Job)
    /         /      \        \
 Child     Chile   Child    Child
* 첫번째 child가 fail되면 다른 child들을 cancel
```

# SupervisonJob
```kotlin
class ExampleClass {
    val scope = CoroutineScope(Job())
}

class ExampleClass {
    val scope = CoroutineScope(SupervisorJob()) // child 하나가 fail되어도 다른 child는 cancel되지 않도록
}
```
```bash
                Scope (Job)
    /         /      \        \
 Child     Chile   Child    Child
* 첫번째 child가 fail되어도 다른 child들을 영향 없음
```

# coroutineScope supervisorScope
 * 어떤 Coroutine 내부에서 다른 scope들을 생성할 수 있다. 이렇게 하면 다른 coroutine들을 논지적인 group으로 묶기가 가능하다.
 * outer scope는 suspend되고 내부에서 모든 coroutine들이 생성될때까지 resume을 진행하지 않는다.
 * SupervisorScopes
   * outer Coroutine에서 사용된 CoroutineContext를 상속
   * 새로운 job이나 SupervisorJob으로 job element를 overwrite

```kotlin
class ExampleRepository {
    suspend fun doSomething() {
        // 병렬로 task들을 실행
        // 하나의 task가 fail되더라도 다른 task들은 계속 진행
        supervisorScope {
            launch { /** task 1 **/ }
            launch { /** task 2 **/ }
        }
    }
}
```
 * 
