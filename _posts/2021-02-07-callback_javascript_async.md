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

오잉, 이게 무슨 일인가??

C 예제 코드에서는 foo가 먼저 출력되고 bar가 출력되었지만, 예상과 달리 JavaScript 예제 코드는 bar가 먼저 출력되고 foo가 이후에 출력된다.
 
더 자세히 살펴보면, C 예제에서는 실행 후 2초 경과시점에 foo가 출력되고 3초 경과시점에 bar가 출력되는데, JavaScript 예제에서는 실행 후 1초 경과시점에 bar가 출력되고 2초 경과시점에 foo가 출력되는 것을 볼 수 있다. 즉, JavaScript에서는 `setTimeout(foo, 2000)` 이 끝날 때까지 기다린 후 `setTimeout(bar, 1000)` 이 실행 되는게 아니라, 마치 실행과 동시에 `setTimeout(foo, 2000)` 와 `setTimeout(bar, 1000)` 이 실행되는 것 같이 작동한다.

도대체 왜 이런 현상이 일어나는 걸까?? 이것을 이해하려면 JavaScript의 비동기성에 대한 이해가 먼저 필요하다.

# Into the weeds of JavaScript - Event Loop & Task Queue

<center><img src="/assets/img/sprint3/eventloop.png" alt="eventloop"></center>
<center><em>image from reference [2]</em></center>

우선, JavaScript의 평가와 실행은 모두 V8과 SpiderMonkey와 같은 JavaScript Engine에 의해서 이루어진다. JavaScript Engine에는 call stack과 heap이 있는데, call stack은 각 함수가 평가되어서 생기는 실행 컨택스트를 순서대로 Last-in-First-out 형태로 실행한다. Heap은 동적으로 생성된 객체들이 존재하는 메모리 공간이라고 생각하면 된다. 아마 call stack과 heap에 대해서 익숙한 독자들이 많을 것으로 예상해서 자세한 설명은 생략하겠다.

위의 예제의 `foo()`, `bar()`, `setTimeout()` 함수들도 모두 실행되기 위해서는 call stack에 하나하나씩 추가되면서 실행된다. 하지만 `setTimeout()` 함수는 종료하기 전에 매개변수로 받은 callback 함수를 곧바로 call stack에 추가하지 않고, 조금 특별하게 작동한다.

브라우저나 Node.js같은 runtime environment는 JavaScript Engine만 포함하고 있는 것은 아니다. 위의 그림에서도 볼 수 있듯이 Web API[3] 라는 것을 제공하는데, `setTimeout()` 은 이 Web API에서 제공되는 함수이다. 이 `setTimeout()` 함수이 실행되면 delay로 받은 밀리초만큼 기다린 후, 실행이 종료되면서 매개변수로 받은 callback 함수를 task queue (aka. callback queue aka. event queue) 라는 공간에 callback 함수의 실행 컨텍스트를 임시로 저장한다. 다시 말해서, `setTimeout()` 에 주어진 callback 함수는 delay 이후 바로 call stack에 push되어 실행되는 것이 아니다.

`setTimeout()` 함수에 의해서 task queue에 등록된 callback 함수의 실행 컨텍스트는 다시 call stack으로 옮겨져야지만 실행된다. 모든 실행 컨텍스트는 JavaScript Engine에 의해서 call stack에서만 실행된다는 점을 잊지 말자. 이 task queue에서 call stack으로 실행 컨텍스트를 옮기는 작업을 담당하는 것이 event loop이다. Event loop는 주기적으로 task queue에 남아있는 작업이 있는지 체크한다. 만약 작업이 남아있고 call stack에는 작업이 없어서 비어있는 상태라면, event loop가 실행 컨택스트를 call stack으로 옮겨서 실행을 시작한다. call stack이 비어있지 않은 상태라면, 빌 때까지 옮기지 않는다.

이름에서도 짐작할 수 있듯이, task queue는 FIFO 구조의 queue이다. `setTimeout()` 가 여러 개 있어서 등록된 callback들이 task queue에 쌓일 수도 있다. 이 때, event loop는 가장 먼저 task queue에 도달한 task를 call stack으로 옮긴다.

<center> . . . </center>

이 배경지식을 토대로 위의 예제 코드를 다시 한 번 살펴보자. 실행 결과는 프로세스 실행 후 bar가 1초 경과시점에, foo가 2초 경과시점에 출력되었었다. 그 이유는, 다음과 같은 순서로 프로세스가 실행되기 때문이다.

1. 첫번째 `setTimeout(foo, 2000)` 이 call stack에 push 되어서 실행된다. `setTimeout()` 은 금방 실행이 끝나고 브라우저 혹은 Node.js의 Web API 컴포넌트에 2초의 타이머를 등록한다.[4]
2. 마찬가지로 `setTimeout(bar, 1000)` 이 call stack에서 실행되며, `setTimeout()` 은 금방 실행을 마치고 Web API 컴포넌트에 1초의 타이머를 등록한다.
3. 실행 후 1초의 시간이 경과 되었을 때, 앞서 등록된 timer에 의해서 `bar()` 의 실행 컨텍스트가 task queue에 등록된다.
4. event loop는 주기적으로 task queue를 체크하면서, 새로 등록된 `bar()` 를 발견한다. 그리고 call stack이 비어있는지 체크한다. 현재로서는 더 이상 실행되고 있는 코드는 없으므로 call stack이 비어있고, event loop는 `bar()` 를 call stack으로 옮겨서 실행되게끔 한다. 그래서 bar가 출력된다.
5. 실행 후 2초의 시간이 경과 되었을 때, 앞서 등록된 timer에 의해서 `foo()` 의 실행 컨택스트가 task queue에 등록된다.
6. 마찬가지로, event loop는 `foo()` 를 task queue에서 call stack으로 옮겨서 실행되게끔 한다. foo가 출력된다.

<center> . . . </center>

C의 `sleep()` 함수는 주어진 delay만큼 프로세스를 중단해서 `sleep()`가 이루어지고 있는 도중에는 그 어떠한 코드도 실행되지 않는다. 이에 비해서, JavaScript의 `setTimeout()` 함수는 주어진 delay에 맞게 callback 함수를 task queue에 등록하는 것을 미룰뿐이지, 그 사이에 다른 코드가 기다리지 않고 실행된다. 이런식으로 코드가 비동기적으로 실행되는 것을 비동기적 프로그래밍 (asynchronous programming)이라고 부른다. `setTimeout()` 함수만 그런 것이 아니다. 그 외의 Web API에서 제공되는 `setInterval()` 함수, HTTP 요청, 이벤트 리서너는 모두 비동기 형태로 이루어진다.

한가지 흥미로운 점은 JavaScript Engine은 단일 스레드로 작동한다는 점이다. 단일 스레드지만 비동기적 기능을 지원을 하려고 task queue와 event loop가 보조로 사용되는 것이다.

event loop와 task queue에 관한 자세한 내용은 <모던 자바스크립트 Deep Dive (이웅모 지음)>[5] 책의 42장을 읽는 것과 [이 유튜브 영상](https://www.youtube.com/watch?v=8aGhZQkoFbQ)[6] 을 추천한다.

<div class="container">
  <center><iframe class="responsive-iframe" src="https://www.youtube.com/embed/8aGhZQkoFbQ" frameborder="0" allowfullscreen></iframe></center>
</div>



# But why..? - Understanding event-driven programming

 이제 우리는 JavaScript의 예제가 비동기적으로 진행되기 때문에 왜 C예제랑 다르게 작동하는지 알고 있다. 하지만 JavaScript의 `setTimeout()` 는 어째서 비동기 형태로 실행되는 것일까?

이 해답을 찾으려면 JavaScript의 유래를 이해해야한다[7]. JavaScript는 애초에 브라우저에 동적인 기능을 추가하려고 탄생한 프로그래밍 언어이다. 브라우저 환경에서 동작하려면 비동기적으로 작동하는 것이 유리/필요하다. 브라우저에서 JavaScript가 어떤 방식으로 이용되는지 잘 생각해보자. 사용자와는 mouse 움직임, keyboard 타이핑과 같은 event를 처리하는데에 사용된다. 또, 서버로 ajax call을 하는 경우에도 서버와 네트워크 I/O로 통신을 하는데 사용된다. 즉, 브라우저는 본질적으로 I/O중심으로 작동한다.

이런 I/O중심의 소프트웨어는 동기적으로 작동하기에는 무리가 있다. 동기적으로 작동하려면, 하나의 I/O request가 끝까지 처리될때까지 아무 일도 못하고 마치 C의 sleep() 함수처럼 기다려야한다. Input request가 도달한 후에 처리되어서 다시 output으로 반환되기까지는 상대적으로 굉장히 긴 시간이 걸린다. 즉, 브라우저의 JavaScript가 동기적으로 작동한다면 하나의 마우스 클릭 event가 끝까지 처리되거나 ajax call이 완료될 때까지 화면을 움직이지도 못하고, 마우스와 키보드도 사용할 수 없게 될 것이다. 그러기 때문에 마우스 클릭, 키보드 타이핑, ajax call과 같은 I/O event가 생성될 때 처리를 나중으로 미루는 비동기적으로 작동하는 것이 성능적인 면에서 중요하다.

JavaScript에서는 각각 I/O event를 동기적으로 처리하지 않고 callback 함수를 등록만 해놓은 후에 나중으로 callback 함수 호출을 미룬다. 이런식으로 하나의 event가 발생할때마다 등록해놓은 callback 함수 호출로 처리하는 방식을 event-driven programming[8]이라고 한다. 처음부터 끝까지 프로세스의 작동이 정의된 것이 아니라, 그 때 그 때마다 들어오는 I/O에 따라서 반응해야 하기 때문에 프로세스에서 callback을 호출하는 Inversion of Control (IoC) 형태로 진행된다.  이러한 event-driven programming은 브라우저 환경외에서도 GUI같은 환경이나, 운영체제의 interprocess communication (IPC)에서 필요한 signal handling과 같은 상황에서도 자주 사용되는 programming paradigm이다.

<center>. . .</center>

우리는 주로 코드가 여러 줄 있으면 한 줄 한 줄씩 차례대로 실행될 것이라고 생각하는 동기형 모델에 익숙해있다. 주로  JavaScript에서는 이러한 비동기형태의 코드 때문에 예상하지 않았던 방식으로 코드가 실행될 수 있기 때문에 더욱 각별한 주의를 요한다. 그렇다고 해서 JavaScript에서만 비동기형태로 코드가 진행된다는 말은 아니다. 프로그래밍 언어와 동기/비동기 형태의 programming paradigm은 서로 다른 개념이라서, C 같은 언어에서도 충분히 비동기 형태로 코드를 짤 수 있다. (궁금하다면 `man select` 와 `man poll` 을 읽어본 후 event-driven server를 찾아보는 것도 권장한다.) 다만, JavaScript는 앞서말한 이유 때문에 본질적으로 asynchronous한 부분이 비교적 강해서, ajax나 event handling등을 할 때 염두해두면 좋지 않을까 싶다.

다음 포스트에는 시간이 된다면 이번 주제에 이어서 다양한 콜백 패턴과 콜백의 단점, Promise, async/await에 대해서 알아볼 예정이다.



## References
[1] https://stackoverflow.com/questions/142789/what-is-a-callback-in-c-and-how-are-they-implemented

[2] https://velog.io/@azurestefan/How-Javascript-works.-Event-loop-Concurrency-Asynchronous-Callback-Callback-Queue-Blocking-etc

[3] https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Client-side_web_APIs/Introduction

[4] https://www.javascripttutorial.net/javascript-bom/javascript-settimeout/#:~:text=Introduction%20to%20JavaScript%20setTimeout()&text=cb%20is%20a%20callback%20function,the%20delay%20defaults%20to%200.

[5] http://www.yes24.com/Product/Goods/92742567

[6] https://www.youtube.com/watch?v=8aGhZQkoFbQ

[7] https://en.wikipedia.org/wiki/JavaScript

[8] https://en.wikipedia.org/wiki/Event-driven_programming
