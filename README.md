# База задач по корутинам — 95% реальных кейсов

Набор задач, покрывающих 95% практики корутин в Android/JVM. Формат: краткая постановка → стартовый код. **Решения вынесены в конец документа** (секция «Решения») и не мешают лайвкодингу.

> **Как использовать**
>
> 1. Для тренировки: сначала смотри постановку и стартовый код. 2. Когда захочешь проверить себя — пролистай в конец к разделу «Решения» и открой нужный номер.

---



## Задача 1. Параллельные запросы + отмена при любой ошибке + состояние через sealed interface

**Постановка.** Два независимых запроса: `fetchUser()` и `fetchUserOrders()`. Выполнить *параллельно*. Если **любо* из них завершается ошибкой — **вся корутинаотменяется*. Состояние описывается через `sealed interface` с тремя состояниями: `Loading`, `Success`, `Failure`. *(См. решение 1 в конце.)*


**Стартовый код**

```kotlin
fun main() {
    runBlocking {
       
    }
}

data class MyUser(
    val id: String,
    val name: String
)

data class Order(
    val id: String,
    val userId: String,
    val orderNumber: Int
)

sealed interface AggregateState {
    data object Loading : AggregateState
    data class Success(val user: MyUser, val orders: List<Order>) : AggregateState
    data class Failure(val error: Throwable) : AggregateState
}

suspend fun fetchUser(id: Int): MyUser { 
    delay(3000)
    return MyUser(
        id = "1",
        name = "Andrew"
    )
}

suspend fun fetchUserOrders(id: Int): List<Order> { 
    delay(6000)
    return listOf(
        Order(
            id = "order1",
            userId = "user1",
            orderNumber = 1,
        ),
        Order(
            id = "order2",
            userId = "user2",
            orderNumber = 2,
        )
    )
}

suspend fun fetchUserError(id: Int): MyUser {
    delay(3000)
    throw RuntimeException("User error")
}

suspend fun fetchUserOrdersError(id: Int): List<Order> {
    delay(6000)
    throw RuntimeException("Order error")
}
```
### Решение

[Перейти к решению 1](#solution-1).

---



## Задача 2. SupervisorJob vs обычный Job (чтение + рефакторинг)

**Проблема.** В коде кидается exception в функции `withExc()` хочется, чтобы корутина могла обработать ошибку и не завершалась. Как работает сейчас? *(См. решение 2 в конце.)*


**Стартовый код**

```kotlin
fun main() {
    val ceh = CoroutineExceptionHandler { _, exc ->
        println("CEH: ${exc.message}")
    }

    runBlocking {
        val cs = CoroutineScope(Dispatchers.Default + SupervisorJob()).launch {
            val one = launch(ceh) {
                withExc()
            }
            val two = launch {
                delay(2000)
                println("end")
            }

            one.join()
            two.join()
        }

        cs.join()
    }
}

suspend fun withExc(): String {
    delay(500)
    throw RuntimeException("withExcException")
}
```

### Решение

[Перейти к решению 2](#solution-2).

---



## Задача 3. «Что будет выведено?» (чтение + рефакторинг)

**Стартовый код** *(См. решение 3 в конце.)*

```kotlin
fun main() {
    runBlocking {        
        println("1")
        println(getW())
        println("2")
    }
}

suspend fun getW(): String = coroutineScope {  
    val f = async { getF() }   
    val t = async { getT() } 
    delay(200)
    t.cancel()
    "${f.await()}"
}

suspend fun getF(): String {
    delay(1000)
    return "Sunny"
}

suspend fun getT(): String {
    delay(100)
    throw AssertionError("T Error")
    return "30 d"
}
```

### Решение

[Перейти к решению 3](#solution-3).


---

## Задача 4. Разделяемый ресурс

**Стартовый код** Что будет выведено? Какими способами можно сделать так, чтобы вывод был 1_000_000?
*(См. решение 4 в конце.)*

```kotlin
fun main() {
    runBlocking(Dispatchers.Default) {
        var counter = 0
        val jobs = List(1_000) {
            launch {
                delay(1000)
                repeat(1_000) {
                    counter++
                }
            }
        }
        jobs.joinAll()
        println("counter=$counter")
    }
}
```


### Решение

[Перейти к решению 4](#solution-4).

---


## Критерии оценки

- Корректность, производительность, надёжность, читаемость, тестируемость.

---

# Решения

> Ниже — эталонные решения.

```kotlin
// Вспомогательная функция логирования (если нужна):
fun Any.t(msg: String) = println("[${'$'}{Thread.currentThread().name}] ${'$'}msg")
```



### Решение 1 (к задаче 1)


```kotlin
fun main() {
    //TODO изначально использовать AggregateState.Loading
    runBlocking {
        val state = getAggregateState()
        println(state)

        val errorState = getAggregateStateWithErrors()
        println(errorState)
    }
}

suspend fun getAggregateState(): AggregateState {
    val scope = CoroutineScope(Dispatchers.IO)

    // top-level корутина, поэтому exception будет только в userDef
    val userDef = scope.async {
        fetchUser(1)
    }

    val ordersDef = scope.async {
        fetchUserOrders(1)
    }

    try {
        val user = userDef.await()
        val orders = ordersDef.await()
        return AggregateState.Success(
            user = user,
            orders = orders,
        )
    } catch (e: CancellationException) {
        // TODO залогировать exception
        throw e
    } catch (e: Exception) {
        // TODO обработать exception
        return AggregateState.Failure(
            error = e
        )
    }
}

suspend fun getAggregateStateWithErrors(): AggregateState {
    val scope = CoroutineScope(Dispatchers.IO)

    // top-level корутина, поэтому exception будет только в userDef
    val userDef = scope.async {
        fetchUserError(1)
    }

    val ordersDef = scope.async {
        fetchUserOrdersError(1)
    }

    try {
        val user = userDef.await()
        val orders = ordersDef.await()
        return AggregateState.Success(
            user = user,
            orders = orders,
        )
    } catch (e: CancellationException) {
        // TODO залогировать exception
        throw e
    } catch (e: Exception) {
        // TODO обработать exception
        return AggregateState.Failure(
            error = e
        )
    }
}

```
**Далее:** [к задаче 2 →](#task-2)

---


### Решение 2 (к задаче 2)

**Стартовый код**
```kotlin
fun main() {
    val ceh = CoroutineExceptionHandler { _, exc ->
        println("CEH: ${exc.message}")
    }

    runBlocking {
        val cs = CoroutineScope(Dispatchers.Default + SupervisorJob()).launch {
            val one = launch(ceh) {
                withExc()
            }
            val two = launch {
                delay(2000)
                println("end")
            }

            one.join()
            two.join()
        }

        cs.join()
    }
}

suspend fun withExc(): String {
    delay(500)
    throw RuntimeException("withExcException")
}

```
Сейчас в изначальном коде обработка ошибок опирается на ceh, который находится в корутине `one`. Обработки ошибок не происходит, так как `one` - **Job**, а не **SupervisorJob** -> обработка должна производиться в скоупе **cs**
Возможное решение: 

```kotlin
fun main() {
    val ceh = CoroutineExceptionHandler { _, exc ->
        println("CEH: ${exc.message}")
    }

    runBlocking {
        val cs = CoroutineScope(Dispatchers.Default + SupervisorJob() + ceh).launch {
            val one = launch() {
                withExc()
            }
            val two = launch {
                delay(2000)
                println("end")
            }

            one.join()
            two.join()
        }

        cs.join()
    }
}

suspend fun withExc(): String {
    delay(500)
    throw RuntimeException("withExcException")
}
```

Проблема повторного запуска кода с **ceh** в том, что **ceh** может только 1 раз обработать ошибку. Поэтому лучше всего - либо обернуть withExc в try-catch блок, либо обернуть весь coroutine scope в этот блок.

```kotlin
fun main() {
    runBlocking {
        val cs = CoroutineScope(Dispatchers.Default + SupervisorJob()).launch {
            val one = launch() {
                try {
                    withExc()
                } catch (e: CancellationException) {
                    throw e
                } catch (e: Exception) {
                    // TODO
                }
            }
            val two = launch {
                delay(2000)
                println("end")
            }

            one.join()
            two.join()
        }

        cs.join()
    }
}

suspend fun withExc(): String {
    delay(500)
    throw RuntimeException("withExcException")
}

```


**Далее:** [к задаче 3 →](#task-3)

---

### Решение 3 (к задаче 3)

**Стартовый код**
```kotlin
fun main() {
    runBlocking {        
        println("1")
        println(getW())
        println("2")
    }
}

suspend fun getW(): String = coroutineScope {  
    val f = async { getF() }   
    val t = async { getT() } 
    delay(200)
    t.cancel()
    "${f.await()}"
}

suspend fun getF(): String {
    delay(1000)
    return "Sunny"
}

suspend fun getT(): String {
    delay(100)
    throw AssertionError("T Error")
    return "30 d"
}
```

Запускаем 2 корутины `f` и `t` параллельно, отменяем корутину `t`, выводим корутину `f`. Проблема в том, что корутина `t` успевает кинуть exception до этого момента.

Вывод:
```kotlin
1
Exception in thread "main" java.lang.AssertionError: T Error
	at ru.wildberries.fintech.c2ctransfer.confirmation.data.sbp.SbpTransactionDataSourceImplKt.getT(SbpTransactionDataSourceImpl.kt:127)
	at ru.wildberries.fintech.c2ctransfer.confirmation.data.sbp.SbpTransactionDataSourceImplKt$getT$1.invokeSuspend(SbpTransactionDataSourceImpl.kt)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
	at kotlinx.coroutines.DispatchedTaskKt.resume(DispatchedTask.kt:221)
	at kotlinx.coroutines.DispatchedTaskKt.dispatch(DispatchedTask.kt:154)
	at kotlinx.coroutines.CancellableContinuationImpl.dispatchResume(CancellableContinuationImpl.kt:470)
	at kotlinx.coroutines.CancellableContinuationImpl.resumeImpl$kotlinx_coroutines_core(CancellableContinuationImpl.kt:504)
	at kotlinx.coroutines.CancellableContinuationImpl.resumeImpl$kotlinx_coroutines_core$default(CancellableContinuationImpl.kt:493)
	at kotlinx.coroutines.CancellableContinuationImpl.resumeUndispatched(CancellableContinuationImpl.kt:596)
	at kotlinx.coroutines.EventLoopImplBase$DelayedResumeTask.run(EventLoop.common.kt:497)
	at kotlinx.coroutines.EventLoopImplBase.processNextEvent(EventLoop.common.kt:263)
	at kotlinx.coroutines.BlockingCoroutine.joinBlocking(Builders.kt:95)
	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking(Builders.kt:69)
	at kotlinx.coroutines.BuildersKt.runBlocking(Unknown Source)
	at kotlinx.coroutines.BuildersKt__BuildersKt.runBlocking$default(Builders.kt:47)
	at kotlinx.coroutines.BuildersKt.runBlocking$default(Unknown Source)
	at ru.wildberries.fintech.c2ctransfer.confirmation.data.sbp.SbpTransactionDataSourceImplKt.main(SbpTransactionDataSourceImpl.kt:105)
	at ru.wildberries.fintech.c2ctransfer.confirmation.data.sbp.SbpTransactionDataSourceImplKt.main(SbpTransactionDataSourceImpl.kt)

Process finished with exit code 1

```


 #### Что делать, если мы хотим вывести `f` в любом случае, а менять и обрабатывать exception внутри `t` мы не можем?

Можно воспользоваться `supervisorScope`, а дальше все зависит вас. Можно, например, воспользоваться функцией `invokeOnCompletion` для удобного вывода ошибки.

```kotlin
fun main() = runBlocking {
    println("1")
    println(getW())
    println("2")
}

suspend fun getW(): String = supervisorScope {
    val f = async { getF() }
    val t = async { getT() }
    t.invokeOnCompletion { e ->
      if (e != null) println("t failed: ${e.message}")
    }
    delay(200)
    t.cancel()
    f.await()
}

suspend fun getF(): String {
    delay(1000)
    return "Sunny"
}

suspend fun getT(): String {
    delay(100)
    throw AssertionError("T Error")
    return "30 d"
}
```


**Далее:** [к задаче 4 →](#task-4)

---

### Решение 4 (к задаче 4)

#### Решение через AtomicInteger
Более подробно посмотреть **почему и как** это работает [тут](https://youtu.be/N3tW9VdZV-A?t=1291)

```kotlin
fun main() {
    runBlocking(Dispatchers.Default) {
        val counter = AtomicInteger(0)
        val jobs = List(1_000) {
            launch {
                delay(1000)
                repeat(1_000) {
                    counter.getAndIncrement()
                }
            }
        }
        jobs.joinAll()
        println("counter=$counter")
    }
}
```

#### Решение через mutex
В потоках стоит использовать альтернативу - synchronized. Аналогичная задача [тут](https://youtu.be/N3tW9VdZV-A?t=1351)

```kotlin
fun main() {
    runBlocking(Dispatchers.Default) {
        var counter = 0
        val mutex = Mutex()
        val jobs = List(1_000) {
            launch {
                delay(1000)
                repeat(1_000) {
                    mutex.withLock {
                        counter++
                    }
                }
            }
        }
        jobs.joinAll()
        println("counter=$counter")
    }
}

```

