---
layout: post
comments: true
title:  "Understanding Callback Functions and JavaScript Async"
date:   2021-02-07 23:04:40 +0900
categories: jekyll update
---

# Why use callback functions?

오늘은 callback과 JavaScript의 비동기성에 대해서 더 자세히 알아보자.

"Callback function"이라 하면 JavaScript의 고유 특성이라고 알고 있는 사람들이 있다. 하지만 실생활에서 callback의 개념을 마주칠 일이 JavaScript에서 흔해서일 뿐이지, callback은 어느 한 프로그래밍 언어의 고유적 특성이 아니다.

Callback function는 다른 코드의 인수로 넘겨져서, 그 코드(caller)가 먼저 실행된 이후에 나중에 실행(call-back)될 수 있도록 하는 함수이다.

예를 들어서 다음과 같은 케이스에 callback 패턴을 유용하게 사용할 수 있다<sp>[1]</sp>. 아래의 코드는 C 언어이다.

```c
/* C */
#include <stdlib.h>
#include <stdio.h>

void populate_array(int *array, size_t arraySize, int (*getNextValue)(void))
{
    for (size_t i=0; i<arraySize; i++)
        array[i] = getNextValue();
}

int getNextRandomValue(void)
{
    return rand();
}

int main(void)
{
    int myarray[10];
    populate_array(myarray, 10, getNextRandomValue);
}
```

일단 `main()` 함수에서 `myarray` 배열을 생성한다. `populate_array()` 에서는 `myarray` 의 각 배열원소에 callback 함수인 `getNextValue()` 가 반환하는 값을 할당한다. 여기서, `getNextValue()` 매개변수는 호출할 때 인수로 쓴 `getNextRandomValue()` 를 사용하게 된다.

Callback 패턴의 장점은 함수를 modular하게 구성해서 필요에 따라서 원하는 함수를 끼워맞춰서 사용할 수 있다는 점이다. 예를 들어서 새로운 배열 `myarray2`에는 `getNextRandomValue()` 를 사용하지 않고 다음과 같은 `getNextIncrValue()` 를 사용해서 배열원소에 값을 할당하고 싶다고 치자.

```c
...

int getNextIncrValue(void)
{
    static int val = 0;
    return val++;
}

...

int main(void)
{
    int myarray[10];
    populate_array(myarray, 10, getNextRandomValue);
    int myarray2[10];
    populate_array(myarray, 10, getNextIncrValue);
}
```

이 경우에 `myarray` 는 `getNextRandomValue()` 가 반환한 값을 할당받게 되고, `myarray2` 는 `getNextIncrValue()` 가 반환한 값을 할당받게 된다.

Callback은 보다싶이 코드의 유연성을 높여줄 수 있는 패턴이다. 특히 함수형 프로그래밍에서 자주 등장하는 패턴인데, 함수형 프로그래밍에 관한 부분은 이 글의 범위를 벗어나므로 다음번에 기회가 되면 더 자세히 다루도록 하겠다.


# A mystery appears...

C 언어로 다음과 같은 callback 예제를 살펴보자.

```c
/* C */
#include <unistd.h>
#include <stdio.h>

void sleepWithCallback(void (*fn)(void), int delay) {
    sleep(delay);
    fn();
}

void foo() {
    printf("foo\n");
}

void bar() {
    printf("bar\n");
}

int main() {
    sleepWithCallback(foo, 2);
    sleepWithCallback(bar, 1);
}
```

`sleepWithCallback()` 는 두 가지 매개변수를 받는다. 두번째 인수는 몇 초 동안 `sleep()` 를 할지 정하고, 첫번째 인수는 `sleep()` 가 끝난 후에 호출할 callback 함수다.

`main()` 에서는 `sleepWithCallback()` 함수를 이용해서 먼저 2초동안 sleep 한 후에 `foo()` 를 호출하고, `foo()` 가 종료하면 또  `sleepWithCallback()` 를 이용해서 1초동안 sleep한 후에 `bar()` 를 호출한다. 이 코드의 예상되는 실행 결과는 2초동안 아무것도 출력되지 않다가 foo가 출력되고, 그에 이어 1초동안 기다렸다가 또 bar가 출력되는 것이다. 총 소요시간은 즉 대략 3초이다. 실제로 컴파일해서 실행해보면 예상대로 실행된다.

이제 자바스크립트로 같은 코드를 짜서 브라우저 혹은 Node.js에서 실행해보자. 브라우저와 Node.js는 앞서 살펴본 `sleepWithCallback()` 과 같은 타이머 함수를 `setTimeout()` 이라는 함수로 직접 제공해주며, 기능은 비슷하게 일정 시간동안 기다린 후에 주어진 callback 함수를 호출하는 함수이다. <sp>(단, `sleepWithCallback()` 은 초(seconds) 단위로 delay를 받지만 `setTimeout()` 은 밀리초 (milliseconds) 단위로 delay를 받는다.)</sp>

```javascript
/* JavaScript */
let foo = () => {
    console.log("foo");
}

let bar = () => {
    console.log("bar");
}

setTimeout(foo, 2000);
setTimeout(bar, 1000);
```
앞서 살펴본 C 예제랑 같은 코드를 단지 JavaScript로 옮긴 코드이기 때문에 같은 방식으로 작동할 것으로 예상한다. 역시나 브라우저 혹은 Node.js의 인터프리터에서 실행해보면 같은 결과가 나온....

오잉, 이게 무슨 일인가?

C 예제 코드에서는 foo가 먼저 출력되고 bar가 출력되었지만, JavaScript 예제 코드는 bar가 먼저 출력되고 foo가 이후에 출력된다..!
 
더 자세히 살펴보면, C 예제에서는 실행 후 2초 경과시점에 foo가 출력되고 3초 경과시점에 bar가 출력되는데, JavaScript 예제에서는 실행 후 1초 경과시점에 bar가 출력되고 2초 경과시점에 foo가 출력되는 것을 볼 수 있다. 즉, JavaScript에서는 `setTimeout(foo, 2000)` 이 끝날 때까지 기다린 후 `setTimeout(bar, 1000)` 이 실행 되는게 아니라, 마치 실행과 동시에 `setTimeout(foo, 2000)` 와 `setTimeout(bar, 1000)` 이 실행되는 것 같이 작동한다.

도대체 왜 이런 현상이 일어나는 걸까?? 이것을 이해하려면 JavaScript의 비동기성에 대한 이해가 먼저 필요하다.

# Into the weeds of JavaScript



## References
[1] https://stackoverflow.com/questions/142789/what-is-a-callback-in-c-and-how-are-they-implemented
