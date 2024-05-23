# Kotlin Coroutines 간단 소개 

## 개요

회사에서 Kotlin을 주력으로 사용하면서 자연스레 coroutine을 종종 사용하게 되었습니다. 한국어로 적으면 코틀린 코루틴이 되는데, 보시다시피 굉장히 실수하기 좋게 생겼습니다. 읽으시기 편하도록 (그리고 제 사적인 편리함을 위해) 두 용어는 영문으로 적겠습니다.

Kotlin에 대한 소개는 앞으로 차차 하도록 하고, 오늘은 coroutine에 대해 간단하게 소개해 드리려고 합니다.

## 코루틴 정의

Coroutine의 역사와 같은 얘기는 스킵하도록 하겠습니다. Coroutine은 무엇인지 간단히 다루어 보겠습니다.

위키피디아에서는 Coroutine을 아래와 같이 정의합니다.

> Coroutines are computer program components that allow execution to be suspended and resumed, generalizing subroutines for cooperative multitasking.

즉,

> Coroutine은 실행 중 중단과 재개가 가능한 컴퓨터 프로그램 구성 요소입니다. 협동적 (cooperative) 멀티태스킹을 위한 서브루틴 관계를 일반화합니다.

Coroutine의 Co는 coop에서 옵니다. 합께 협업한다는 의미를 나타내죠. Routine은 무언가 일을 처리하는 일련의 과정을 뜻합니다. 우리가 아침에 일어나서, 세수를 하고, 출근을 해서 일을 하고... 의 과정을 daily routine이라고 할 수 있지요.

## 코드 예시 (기존)

우리의 언어로 (사실 아마 저한테만 익숙한 언어로) 루틴을 떠올려 보면,

```kotlin
fun dailyRoutine(me: Human = katfun) {
  me.wakeup()
  me.washMyFace()
  me.goToPangyo()
  work(target = me, time = 8)	 // this actually takes 8 hours. probably.
  me.goHome()
  preparePresentation(target = me, time = 1)  // and this takes exactly an hour.
  me.playGames()
  me.sleep()
}

private fun work(target: Human, time: Int)

private fun preparePresentation(target = Human, time: Int)
```

하루 루틴을 이렇게 표현할 수 있습니다. 설명의 편의를 위해 `work`, `preparePresentation` 함수는 I/O 작업과 같다고 간주하겠습니다.

이 루틴은 **하나의 스레드 위에서** 이루어집니다. 즉, `dailyRoutine`은 하나의 스레드 위에서 시작해서, 위에서부터 차례대로 일을 처리합니다. 중요한 점은, `work`와 `preparePresentation`이 호출되었을 때, 즉 외부 I/O가 완료되기를 기다리는 동안, `dailyRoutine`이 실행되고 있는 스레드는 **블로킹**됩니다.

블로킹? 쉽게 말하면, 호출된 작업이 처리 완료되도록 `dailyRoutine`이 스레드를 잡고 안 놔준다는 말입니다. 8시간 동안, 1시간 동안.

그러던 중, 동일한 `Human` 타입의 `vsfe`가 `dailyRoutine`을 따라가고 싶다면, 어떻게 될까요? (싱글 스레드임을 가정)

```kotlin
fun main() {
  // katfun, vsfe are initialised
  dailyRoutine(katfun) // takes quite a long time, mostly because of I/O tasks
  dailyRoutine(vsfe)	// should wait until previous routine gets done
}
```

`dailyRoutine(vsfe)`는 `dailyRoutine(katfun)`이 전부 처리될 때까지 기다려야 합니다. 막상 `dailyRoutine(katfun)`은 대분의 시간을 시켜 둔 I/O 작업이 끝나길 기다리며 보냅니다.

위의 예시는 Coroutine을 사용하지 않은, 기존의 경우 발생하는 일입니다. 외부 작업의 결과를 기다리느라 점유한 스레드가 놀고 있는 점이 너무 아쉽습니다.

## 코드 예시 (Coroutine 사용)

Coroutine은 이런 관점에서, "루틴을 중단하고 재개할 수 있다"는 접근법을 제공합니다. 중단하고 재개한다는 것이 무엇일까요?

```kotlin
@Test
fun main() = runBlocking {
    // katfun, vsfe are initialised
    launch { dailyRoutine(katfun) }
    launch { dailyRoutine(vsfe) }
}

suspend fun dailyRoutine(me: Human) {
  me.wakeup()
  me.washMyFace()
  me.goToPangyo()
  work(target = me, time = 8)	 // the routine "SUSPENDS" from here until work is done
  me.goHome()
  preparePresentation(target = me, time = 1)  // the routine "SUSPENDS" from here until prepare... is done
  me.playGames()
  me.sleep()
}
```



## 스레드와의 비교

suspend는 중단한다는 의미를 가집니다. 루틴이 진행되다가 중단되는 시점을 만나면 (예: I/O 작업 호출), **루틴은 스레드를 스레드 풀에 반환**합니다. 반환된 스레드는 또 다른 시작 가능한 루틴을 실행시킬 수 있습니다.

위의 예시에서는, 1개 스레드임을 전제했을 때, 아래와 같은 순서가 되겠습니다.

* `dailyRoutine(katfun)` 실행
* `katfun.work()`를 만났을 때, `dailyRoutine(katfun)`은 중단되며, 해당 작업은 heap에 저장됩니다. 스레드가 반환됩니다.
  * 어떻게 중단된 suspend function이 다시 실행되는지는... 스킵!
* `dailyRoutine(vsfe)`가 실행될 준비 완료인 상태이므로, 반환된 스레드를 가져와서 실행
* `vsfe.work()`를 만났을 때, `dailyRoutine(vsfe)`는 중단
* `dailyRoutine(katfun)`이 중단된 시점부터 재개, `preparePresenation(katfun)`에서 다시 중단
* `dailyRoutine(vsfe)`이 중단된 시점부터 재개, `preparePresenation(vsfe)`에서 다시 중단
* `dailyRoutine(katfun)`이 중단된 시점부터 재개, 루틴을 마치고 종료
* `dailyRoutine(vsfe)`이 중단된 시점부터 재개, 루틴을 마치고 종료

따라서 위와 같은 경우에는 사람이 한 명이든, 두 명이든, 백 명이든! 실제로 처리 완료에 걸리는 시간은 매우 적게 차이나게 됩니다.

한 줄로 요약하면,

> 스레드는 비싸다. 낭비되는 순간 없이 최대한으로 쥐어짜자!

## 예시 코드

코드는 [여기서](https://github.com/kchung1995/Tech-Share-Sample-Codes/blob/main/src/test/kotlin/com/katfun/tech/share/sample/codes/week02/Week02CoroutinesThreads.kt) 볼 수 있어요.

테스트를 위해 아래와 같이 조건을 설정합니다.

* 사용 스레드: 1개
* 코루틴을 중단하는 경우  `delay()` 사용
* 스레드를 블로킹하는 경우 `Thread.sleep()` 사용

코드는 아래와 같습니다.

```kotlin
class Week02CoroutinesThreads {
    @Test
    fun testMain() = runTest {
        val multiThreadDispatcher = newFixedThreadPoolContext(nThreads = 1, name = "multiThreadDispatcher")

        launch(multiThreadDispatcher) { dailyRoutine("katfun") }
        launch(multiThreadDispatcher) { dailyRoutine("vsfe") }
    }

    suspend fun dailyRoutine(text: String) {
        val delayTime = (3..5).random()
        delay(delayTime * 1000L)
//        Thread.sleep(delayTime * 1000L)

        println("Daily routine of $text took $delayTime seconds.")
    }
}
```

앞의 예시와 같은 구조를 가집니다. 루틴이라고 부르던 `dailyRoutine` 내에서, 함수를 중단하거나, 스레드를 블로킹했을 때의 결과를 비교해 봅니다.