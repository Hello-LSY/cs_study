## StreamAPI, Effectively Final

보통 SreamAPI 사용시 외부 변수를 람다식 안에서 사용할 때, final 변수를 사용하라는 얘기를 많이 들은 것 같다.

정확히는 람다 캡처링 시, 람다식 내에 외부 지역 변수를 사용할 때 Effectivel final, 또는 final 변수여야 한다.

Effectively final란 참조를 하더라도 값이 변경되지 않음을 보장하는 컨디션을 일컫는다.

## Capturing lambda, Non-Capturing lambda

람다식에는 두 가지 방식의 람다식으로 나뉘는데, 나뉘는 기준이 간단하다.

-   Capturing Lambda
    -   외부 지역 변수 사용 시

```java
int count = 0;
Runnable runnable = () -> {
	count += 1;
}
```

-   Non-Capturing Lambda
    -   외부 지역 변수 사용 안하면 됨.

```java
Runnable runnable = () -> {
	System.out.println("Non capturing lambda");
}
```

## What is Lambda Capturing?

람다 내부에서 사용하는 외부 변수는 실제 외부 변수를 참조하지 않고 그 값을 복사하여 사용한다.

여기서 왜 그 값을 복사해야 하냐면, , , 지역 변수 `int count = 0;` 를 사용하고 있는 스레드를 `“thread A”`라고 하자. 그리고 Parallel Stream이나 멀티 스레드 환경에서 `“thread B”` 가 람다식이 실행 된다면 사용하려는 count 변수는 어디에서 가져다 써야할까?

count 변수는 `thread A` 의 스택에 변수로 저장되어 있다. 하지만 람다식을 실행시키는 `thread B` 의 스택엔 count 변수가 없다. 스레드 간 스택은 독자적인 영역을 갖기 때문.

그렇기 때문에 Lambda Capturing이 필요함.

예를 들어 다음과 같은 코드가 있다고 생각해보자.

```java
public class ThreadExample {
    public static void main(String[] args) {
        int count = 0;  // Thread A의 스택에 있는 지역 변수

        // Thread B에서 실행될 람다식
        Runnable task = () -> {
            System.out.println(count);  // 다른 스레드의 스택에 있는 변수를 어떻게 접근?
        };

        new Thread(task).start();  // Thread B 시작
    }
}
```

count 변수는 Thread A의 스택에 저장되어 있다. 하지만 람다식을 실행하는 Thread B의 스택에는 count 변수가 없다. 이는 스레드 간 스택이 독자적인 영역을 가지기 때문이다.

이 문제를 해결하기 위해 Lambda Capturing이라는 개념이 필요하다. 위 코드에서 main 메소드는 Thread A에서 실행되며, count 변수는 Thread A의 스택에 저장된다. 새로 생성된 Thread B는 독립적인 스택 영역을 가지므로 Thread A의 지역 변수에 직접 접근할 수 없다.

Java는 Lambda Capturing을 통해 외부 변수의 값을 복사하여 사용한다. 이때 변수는 반드시 final 또는 effectively final이어야 한다.

Parallel Stream, 또는 멀티 스레드 환경에서 참조하고 있는 외부 변수의 값이 바뀌면 이를 알아 차릴 수 없어

Race Condition이 발생할 수 있다. 그렇기 때문에 외부 변수의 값을 변경하려고 하면 컴파일 단계에서 에러를 출력한다.
