---
title: Coroutine에서의 error handling
categories:
  - Android
tags:
  - Android
  - Coroutines
  - Error Handling
  - Exception Handling
---

안드로이드에서는 어느 곳에서도 catch되지 않은 uncaught exception이 발생하면 기본적으로 앱이 강제 종료되는 crash가 발생한다.  
유저 입장에서는 열심히 정보를 입력하고 마지막 단계까지 왔는데, 갑자기 앱이 꺼지고 입력했던 정보들마저 모두 날아가버리는 상황이 발생할 수 있는 것이다.  

![비정상 종료](https://developer.android.com/static/topic/performance/images/crash-example-framed.png?hl=ko)  
*출처: https://developer.android.com/games/optimize/crash*

이로 인해 유저는 불쾌함을 느끼며 해당 앱에 대한 신뢰를 잃게 된다. 이렇듯 crash는 유저 이탈의 주된 원인이 된다.  
따라서 에러를 알맞게 처리하는 일은 상당히 중요하다.  
이 글에서는 안드로이드에서 coroutine을 사용할 때의 exception handling 방법에 대해 알아본다.  

## try-catch

당연히 coroutine 안에서도 `try-catch`를 통한 exception handling이 가능하다.  

```kotlin
coroutineScope.launch {
    try {
        throw RuntimeException()
    } catch (e: Exception) {
        println("Caught exception")
    }
}
```  

아래처럼 하나의 `CoroutineScope` 안에 두 개 이상의 coroutine이 존재할 때도 동일하다.  
exception이 `try-catch` 안에서 처리되므로, `launch`까지 exception이 전파되지 않기 때문에 다른 coroutine에 영향을 주지 않는다.  

```kotlin
coroutineScope.launch {
    launch {
        try {
            throw RuntimeException()
        } catch (e: Exception) {
            println("Caught exception 1")
        }
    }
    launch {
        try {
            throw RuntimeException()
        } catch (e: Exception) {
            println("Caught exception 2")
        }
    }
}
```

`try-catch`를 이용한 방법의 장점은 다음과 같다.  
1. `CoroutineScope` 내 여러 개의 coroutine에서 발생한 exception을 각기 다른 동작으로 처리할 수 있다.
2. 하나의 coroutine 내에서도 라인별로 각기 다른 동작으로 처리할 수 있다.

단점은 다음과 같다.  
1. 모든 coroutine에 `try-catch`를 넣어야 하므로 누락 가능성이 높아진다. 누락된 지점에서 exception이 발생하면 crash로 이어진다.
2. 하나의 coroutine에서 exception이 발생한다고 해도 같은 `CoroutineScope`에 위치한 다른 coroutine은 계속 실행된다. 따라서 하나의 트랜잭션으로 묶여야 하는 coroutine 간에는 사용할 수 없다.

## launch vs. async

앞서 언급했던 단점 1번을 조금이나마 보완하기 위해 exception을 한 번에 처리하고자 아래와 같이 작성하려고 할 수 있다.  

```kotlin
coroutineScope.launch {
    try {
        joinAll(
            launch {
                throw RuntimeException()
            },
            launch {
                throw RuntimeException()
            }
        )
    } catch (e: Exception) {
        println("Caught exception")
    }
}
```

실행해보면 exception이 catch문에 잡혀 로그가 찍혔는데도 불구하고 crash가 발생한다.  
`launch`로 생성된 coroutine 내에서 발생한 모든 exception은 parent coroutine에 위임하며, 이를 root coroutine에 도달할 때까지 반복한다.  
따라서 `launch`를 try-catch로 감싸도 parent coroutine인 `coroutineScope.launch()`에 exception이 전파되는 것을 막을 수는 없다.  
async 역시 `launch`와 같은 coroutine builder이지만, `launch`와는 다르게 exception을 parent로 전파하지 않는다.  

```kotlin
coroutineScope.launch {
    try {
        joinAll(
            async {
                throw RuntimeException()
            },
            async {
                throw RuntimeException()
            }
        )
    } catch (e: Exception) {
        println("Caught exception")
    }
}
```

위 코드는 exception이 catch문에 잡혀 로그가 찍히고, crash 없이 안전하게 종료된다. async는 exception을 자동으로 전파하지 않고, 개발자가 직접 처리할 수 있도록 한다.

## CoroutineExceptionHandler

try-catch 외에 exception을 처리할 수 있는 또 다른 방법으로 `CoroutineExceptionHandler`가 있다. `CoroutineExceptionHandler`를 사용하면 coroutine 내에서 발생한 uncaught exception을 처리하는 기본 동작을 정의할 수 있다.  

```kotlin
val handler = CoroutineExceptionHandler { _, throwable ->
    println("error occurred: ${throwable.message}")
}
coroutineScope.launch(handler) {
    throw RuntimeException()
}
```

`CoroutineExceptionHandler`를 root coroutine에 전달하면, child coroutine 내에서 발생하는 모든 uncaught exception을 한 곳에서 처리할 수 있다.  
주의할 점은 `CoroutineExceptionHandler`를 root coroutine이 아닌 child coroutine에 전달하면 아무런 효과가 없다는 것이다.  
앞서 설명했듯이, 모든 coroutine은 exception을 자동으로 전파하기 때문에 child coroutine의 `CoroutineExceptionHandler`에서 처리하지 않고 parent coroutine에 처리를 위임한다.  
따라서 root가 아닌 coroutine에 정의된 `CoroutineExceptionHandler`는 의미 없는 코드가 된다.  

`CoroutineExceptionHandler`를 사용하는 방법의 장점은 다음과 같다.  
1. 여러 개의 coroutine에서 발생하는 exception을 한 곳에서 처리할 수 있다.
2. `CoroutineExceptionHandler`를 각 목적에 맞게 구현 및 분리하여 여러 곳에서 재사용할 수 있다.

단점은 다음과 같다.  
1. root coroutine 하나에만 적용 가능하기 때문에 child coroutine별로 다른 exception handling 동작을 정의하려는 곳에서는 부적절하다.

두 방법 모두 장단점이 존재하기 때문에 상황에 맞게 혼용해서 쓰는 것이 가장 적절하다.  
기본적으로 자주 사용되는 동작을 `CoroutineExceptionHandler`로 구현하여 분리해두고 모든 root coroutine에 적용하되, 기본 에러 처리 동작과 다르게 처리해야 한다면 해당 부분만 `try-catch`를 이용하는 식으로 작성할 수 있다.