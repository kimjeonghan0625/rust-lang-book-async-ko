## Async로 동시성 적용하기

<!-- Old headings. Do not remove or links may break. -->

<a id="concurrency-with-async"></a>

이 섹션에서는 16장에서 스레드로 다루었던 동시성 문제들 중 일부에 async를 적용해 보겠습니다. 이미 앞에서 주요 개념들을 많이 다루었으므로, 이 섹션에서는 스레드와 future 간의 차이점에 집중하겠습니다.

많은 경우, async를 사용한 동시성 작업을 위한 API는 스레드를 사용할 때와 매우 비슷합니다. 하지만 어떤 경우에는 상당히 다르기도 합니다. 스레드와 async의 API가 겉보기에는 비슷해 보여도, 실제 동작 방식은 종종 다르며, 거의 항상 성능 특성도 다릅니다.

<!-- Old headings. Do not remove or links may break. -->

<a id="counting"></a>
### `spawn_task`로 새로운 태스크 생성하기

[스폰으로 새로운 스레드 생성하기][thread-spawn]<!-- ignore -->에서 처음으로 다뤘던 작업은 두 개의 별도 스레드에서 카운트 업을 하는 것이었습니다. 이번에는 async를 사용해 같은 작업을 해보겠습니다. `trpl` 크레이트는 `thread::spawn` API와 매우 비슷한 `spawn_task` 함수와, `thread::sleep` API의 async 버전인 `sleep` 함수를 제공합니다. 이 둘을 함께 사용해서 Listing 17-6과 같이 카운팅 예제를 구현할 수 있습니다.

<Listing number="17-6" caption="메인 태스크가 다른 내용을 출력하는 동안 새로운 태스크를 생성하여 출력하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-06/src/main.rs:all}}
```

</Listing>

출발점으로, 우리는 `main` 함수에서 `trpl::run`을 사용하여 최상위 함수가 async가 되도록 설정합니다.

> 참고: 이 시점부터 이 장의 모든 예제는 `main`에서 `trpl::run`으로 감싸는 동일한 코드를 포함합니다. 따라서 이후에는 `main`과 마찬가지로 이 코드를 종종 생략할 것입니다. 하지만 실제 코드에는 반드시 포함해야 합니다!

그런 다음, 해당 블록 안에 각각 `trpl::sleep` 호출이 들어 있는 두 개의 루프를 작성합니다. 각 루프는 다음 메시지를 보내기 전에 0.5초(500밀리초) 동안 대기합니다. 한 루프는 `trpl::spawn_task`의 본문에, 다른 하나는 최상위 `for` 루프에 넣습니다. 또한 `sleep` 호출 뒤에 `await`도 추가합니다.

이 코드는 스레드 기반 구현과 유사하게 동작합니다. 실제로, 실행할 때 메시지가 터미널에 표시되는 순서가 매번 다를 수도 있습니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
hi number 1 from the second task!
hi number 1 from the first task!
hi number 2 from the first task!
hi number 2 from the second task!
hi number 3 from the first task!
hi number 3 from the second task!
hi number 4 from the first task!
hi number 4 from the second task!
hi number 5 from the first task!
```
이 버전은 메인 async 블록의 본문에 있는 `for` 루프가 끝나면 바로 종료됩니다. 이는 `spawn_task`로 생성한 태스크가 `main` 함수가 끝날 때 함께 종료되기 때문입니다. 태스크가 끝날 때까지 모두 실행되도록 하려면, 조인 핸들(join handle)을 사용해 첫 번째 태스크가 완료될 때까지 기다려야 합니다. 스레드에서는 `join` 메서드를 사용해 스레드가 끝날 때까지 “블록”했듯이, Listing 17-7에서는 `await`를 사용해 동일한 동작을 할 수 있습니다. 태스크 핸들 자체가 future이기 때문입니다. 이 future의 `Output` 타입은 `Result`이므로, `await`한 뒤에 `unwrap`도 해줍니다.

<Listing number="17-7" caption="조인 핸들에 `await`를 사용해 태스크를 끝까지 실행하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-07/src/main.rs:handle}}
```

</Listing>

이 수정된 버전은 *두* 루프가 모두 끝날 때까지 실행됩니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
hi number 1 from the second task!
hi number 1 from the first task!
hi number 2 from the first task!
hi number 2 from the second task!
hi number 3 from the first task!
hi number 3 from the second task!
hi number 4 from the first task!
hi number 4 from the second task!
hi number 5 from the first task!
hi number 6 from the first task!
hi number 7 from the first task!
hi number 8 from the first task!
hi number 9 from the first task!
```

지금까지 살펴본 바로는 async와 스레드는 기본적으로 동일한 결과를 제공합니다. 단지 문법만 다를 뿐입니다. 즉, 조인 핸들에서 `join`을 호출하는 대신 `await`를 사용하고, `sleep` 호출에도 `await`를 붙입니다.

더 큰 차이점은, 이를 위해 별도의 운영체제 스레드를 생성할 필요가 없다는 점입니다. 사실, 여기서는 태스크조차 생성할 필요가 없습니다. async 블록은 익명 future로 컴파일되므로, 각 루프를 async 블록에 넣고 런타임이 `trpl::join` 함수를 사용해 둘 다 완료될 때까지 실행하도록 할 수 있습니다.

[조인 핸들을 사용해 모든 스레드가 끝날 때까지 기다리기][join-handles]<!-- ignore --> 섹션에서는 `std::thread::spawn`을 호출할 때 반환되는 `JoinHandle` 타입의 `join` 메서드를 사용하는 방법을 보여주었습니다. `trpl::join` 함수는 future를 위한 것으로, 이와 유사하게 동작합니다. 두 개의 future를 전달하면, 두 future가 _모두_ 완료될 때 각각의 출력을 튜플로 담아 반환하는 새로운 future를 생성합니다. 따라서 Listing 17-8에서는 `trpl::join`을 사용해 `fut1`과 `fut2`가 모두 끝날 때까지 기다립니다. 우리는 `fut1`과 `fut2`를 각각 await하지 않고, `trpl::join`이 만들어낸 새로운 future를 await합니다. 반환값은 두 개의 단위값을 담은 튜플이므로, 별도로 사용하지 않습니다.

<Listing number="17-8" caption="두 개의 익명 future를 기다리기 위해 `trpl::join` 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-08/src/main.rs:join}}
```

</Listing>

이 코드를 실행하면 두 future가 모두 끝날 때까지 실행되는 것을 볼 수 있습니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
hi number 1 from the first task!
hi number 1 from the second task!
hi number 2 from the first task!
hi number 2 from the second task!
hi number 3 from the first task!
hi number 3 from the second task!
hi number 4 from the first task!
hi number 4 from the second task!
hi number 5 from the first task!
hi number 6 from the first task!
hi number 7 from the first task!
hi number 8 from the first task!
hi number 9 from the first task!
```

이제는 매번 *정확히 같은* 순서로 결과가 출력되는 것을 볼 수 있습니다. 이는 스레드에서 보았던 것과는 매우 다릅니다. 그 이유는 `trpl::join` 함수가 *공정(fair)* 하게 동작하기 때문입니다. 즉, 각 future를 동일하게 자주 확인하며, 번갈아가며 처리하고, 한 future가 준비되어 있다고 해서 다른 future보다 앞서 나가도록 두지 않습니다. 스레드에서는 운영체제가 어떤 스레드를 확인할지, 얼마나 오래 실행할지 결정합니다. 반면, async Rust에서는 런타임이 어떤 태스크를 확인할지 결정합니다. (실제로는 async 런타임이 동시성 관리를 위해 내부적으로 운영체제 스레드를 사용할 수도 있기 때문에, 공정성을 보장하는 것이 더 복잡해질 수 있지만, 여전히 가능합니다!) 런타임이 모든 연산에 대해 반드시 공정성을 보장해야 하는 것은 아니며, 종종 공정성 여부를 선택할 수 있는 다양한 API를 제공합니다.

아래와 같은 변형을 시도해 보며 future를 await할 때 어떤 결과가 나오는지 확인해 보세요:

- 두 루프 중 하나 또는 둘 다에서 async 블록을 제거해 보세요.
- 각 async 블록을 정의한 직후에 바로 await해 보세요.
- 첫 번째 루프만 async 블록으로 감싸고, 두 번째 루프 본문 뒤에 해당 future를 await해 보세요.

추가 도전 과제로, 각 경우에 코드를 실행하기 *전*에 어떤 출력이 나올지 스스로 예측해 보세요!

<!-- Old headings. Do not remove or links may break. -->

<a id="message-passing"></a>
### 메시지 패싱을 사용한 두 태스크의 카운트 업

future 간에 데이터를 공유하는 방법도 익숙할 것입니다. 이번에도 메시지 패싱을 사용할 것이지만, 이번에는 타입과 함수의 async 버전을 사용합니다. [스레드 간 데이터 전달을 위한 메시지 패싱 사용하기][message-passing-threads]<!-- ignore -->에서와는 약간 다른 접근을 통해, 스레드 기반 동시성과 future 기반 동시성의 주요 차이점을 보여주겠습니다. Listing 17-9에서는 별도의 태스크를 생성하지 않고, 단일 async 블록만 사용해 시작합니다. 이는 별도의 스레드를 생성했던 이전과는 다릅니다.

<Listing number="17-9" caption="async 채널을 생성하고 두 부분을 `tx`와 `rx`에 할당하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-09/src/main.rs:channel}}
```

</Listing>

여기서는 `trpl::channel`을 사용합니다. 이는 16장에서 스레드와 함께 사용했던 다중 생산자, 단일 소비자 채널 API의 async 버전입니다. async 버전의 API는 스레드 기반 버전과 거의 비슷하지만, 리시버 `rx`가 불변이 아닌 가변 참조를 사용하고, `recv` 메서드는 값을 직접 반환하는 대신 await해야 하는 future를 생성한다는 점이 다릅니다. 이제 우리는 송신자에서 수신자로 메시지를 보낼 수 있습니다. 별도의 스레드나 태스크를 생성할 필요 없이, 단순히 `rx.recv` 호출을 await하면 됩니다.

동기식 `std::mpsc::channel`의 `Receiver::recv` 메서드는 메시지를 받을 때까지 블록합니다. 반면, `trpl::Receiver::recv` 메서드는 async이기 때문에 블록하지 않습니다. 메시지를 받거나 채널의 송신 측이 닫힐 때까지 런타임에 제어권을 넘깁니다. 반대로, `send` 호출은 await하지 않습니다. 블록하지 않기 때문입니다. 우리가 메시지를 보내는 채널이 버퍼 제한이 없기 때문에 await할 필요가 없습니다.

> 참고: 이 모든 async 코드는 `trpl::run` 호출의 async 블록 안에서 실행되기 때문에, 그 내부에서는 블로킹을 피할 수 있습니다. 하지만 그 *외부*의 코드는 `run` 함수가 반환될 때까지 블록됩니다. 이것이 바로 `trpl::run` 함수의 핵심입니다. 즉, 어떤 async 코드 집합에서 블록할지, 그리고 어디서 동기 코드와 async 코드 간 전환이 일어날지 *직접 선택*할 수 있게 해줍니다. 대부분의 async 런타임에서 `run`은 실제로 이러한 이유로 `block_on`이라는 이름을 사용합니다.

이 예제에서 두 가지를 주목하세요. 첫째, 메시지는 즉시 도착합니다. 둘째, 여기서 future를 사용하긴 했지만 아직 동시성은 없습니다. 이 리스트의 모든 동작은 future가 전혀 없을 때와 마찬가지로 순차적으로 실행됩니다.

첫 번째 부분을 개선하기 위해, 여러 개의 메시지를 보내고 그 사이에 잠시씩 대기하는 코드를 Listing 17-10과 같이 작성해 보겠습니다.

<!-- We cannot test this one because it never stops! -->

<Listing number="17-10" caption="async 채널을 통해 여러 메시지를 보내고, 각 메시지 사이에 `await`로 대기하기" file-name="src/main.rs">

```rust,ignore
{{#rustdoc_include ../listings/ch17-async-await/listing-17-10/src/main.rs:many-messages}}
```

</Listing>

메시지를 보내는 것뿐만 아니라, 이제 메시지를 받아야 합니다. 이 경우에는 몇 개의 메시지가 들어올지 알고 있으므로, `rx.recv().await`를 네 번 호출해서 직접 받을 수도 있습니다. 하지만 실제 상황에서는 대개 몇 개의 메시지가 올지 *알 수 없으므로*, 더 이상 메시지가 없다는 것을 알 때까지 계속 기다려야 합니다.

Listing 16-10에서는 동기 채널에서 받은 모든 항목을 처리하기 위해 `for` 루프를 사용했습니다. 하지만 Rust에는 아직 *비동기*로 들어오는 항목들에 대해 `for` 루프를 쓸 수 있는 방법이 없으므로, 여기서는 아직 보지 못한 루프인 `while let` 조건 루프를 사용해야 합니다. 이 루프는 [*if let*과 *let else*를 활용한 간결한 제어 흐름][if-let]<!-- ignore -->에서 봤던 `if let` 구문의 루프 버전입니다. 지정한 패턴이 값과 계속 매칭되는 한 루프가 실행됩니다.

`rx.recv` 호출은 future를 생성하며, 우리는 이를 await합니다. 런타임은 future가 준비될 때까지 일시 중지합니다. 메시지가 도착하면 future는 메시지가 도착한 횟수만큼 `Some(message)`로 resolve됩니다. 채널이 닫히면, 메시지가 *하나도 오지 않았더라도* future는 더 이상 값이 없음을 나타내기 위해 `None`으로 resolve되어, 폴링(즉, await)도 중단해야 함을 알려줍니다.

`while let` 루프는 이 모든 과정을 하나로 묶어줍니다. `rx.recv().await`의 결과가 *Some(message)* 이면, 메시지에 접근할 수 있고, *if let*에서처럼 루프 본문에서 사용할 수 있습니다. 결과가 *None*이면 루프가 종료됩니다. 루프가 한 번 돌 때마다 다시 await 지점에 도달하므로, 런타임은 또다시 메시지가 도착할 때까지 일시 중지합니다.

이제 코드는 모든 메시지를 성공적으로 보내고 받을 수 있습니다. 하지만 여전히 몇 가지 문제가 남아 있습니다. 첫째, 메시지가 0.5초 간격으로 도착하지 않고, 프로그램을 시작한 뒤 2초(2,000밀리초)가 지나 *한꺼번에* 도착합니다. 둘째, 이 프로그램은 종료되지 않고, 새로운 메시지를 영원히 기다립니다! <span class="keystroke">ctrl-c</span>로 강제 종료해야 합니다.

우선, 왜 메시지가 각 메시지 사이에 지연이 있는 것이 아니라, 전체 지연이 끝난 뒤 한 번에 도착하는지 살펴봅시다. 하나의 async 블록 안에서는 코드에 등장하는 `await` 키워드의 순서가 실제 프로그램 실행 시점의 실행 순서와 동일합니다.

Listing 17-10에는 async 블록이 하나뿐이므로, 그 안의 모든 코드는 순차적으로 실행됩니다. 여전히 동시성이 없는 셈입니다. 모든 `tx.send` 호출이 실행되고, 그 사이에 모든 `trpl::sleep` 호출과 await 지점이 끼어 있습니다. 그리고 나서야 `while let` 루프가 `recv` 호출의 await 지점을 통과할 수 있습니다.

우리가 원하는 동작, 즉 각 메시지 사이에 sleep 지연이 발생하도록 하려면, `tx`와 `rx` 작업을 각각 별도의 async 블록에 넣어야 합니다. Listing 17-11에서처럼 말이죠. 그러면 런타임이 counting 예제에서처럼 `trpl::join`을 사용해 각각을 독립적으로 실행할 수 있습니다. 여기서도 개별 future가 아니라 `trpl::join` 호출의 결과를 await합니다. 만약 개별 future를 순서대로 await하면, 다시 순차적인 흐름으로 돌아가게 되며, 이는 우리가 *원하지 않는* 동작입니다.

<!-- We cannot test this one because it never stops! -->

<Listing number="17-11" caption="`send`와 `recv`를 각각 별도의 `async` 블록으로 분리하고, 해당 블록의 future를 await하기" file-name="src/main.rs">

```rust,ignore
{{#rustdoc_include ../listings/ch17-async-await/listing-17-11/src/main.rs:futures}}
```

</Listing>

Listing 17-11의 코드로 업데이트하면, 메시지가 2초가 지난 뒤 한꺼번에 출력되는 대신 500밀리초 간격으로 출력됩니다.

하지만 프로그램은 여전히 종료되지 않습니다. 이는 `while let` 루프와 `trpl::join`의 상호작용 때문입니다:

- `trpl::join`이 반환하는 future는 전달된 *두* future가 *모두* 완료되어야만 완료됩니다.
- `tx` future는 `vals`의 마지막 메시지를 보낸 뒤 sleep이 끝나면 완료됩니다.
- `rx` future는 `while let` 루프가 끝나야만 완료됩니다.
- `while let` 루프는 `rx.recv`를 await한 결과가 *None*이 될 때까지 계속됩니다.
- `rx.recv`를 await하면, 채널의 반대쪽 끝이 닫혀야만 *None*을 반환합니다.
- 채널은 우리가 `rx.close`를 호출하거나, 송신자 쪽인 `tx`가 drop될 때에만 닫힙니다.
- 우리는 어디에서도 `rx.close`를 호출하지 않고, `tx`는 `trpl::run`에 전달된 최상위 async 블록이 끝나야 drop됩니다.
- 그런데 이 블록은 `trpl::join`이 완료될 때까지 대기하므로 끝날 수 없습니다. 이로 인해 다시 처음으로 돌아가게 됩니다.

`rx.close`를 호출해서 수동으로 `rx`를 닫을 수도 있지만, 그다지 적절한 방법은 아닙니다. 임의의 개수의 메시지만 처리하고 중단하면 프로그램이 종료되긴 하겠지만, 그 과정에서 메시지를 놓칠 수 있습니다. 우리는 `tx`가 함수가 끝나기 *전에* drop되도록 보장할 다른 방법이 필요합니다.

현재 메시지를 보내는 async 블록은 `tx`를 빌려서 사용하고 있습니다. 메시지를 보내는 데 소유권이 필요하지 않기 때문입니다. 하지만 만약 `tx`를 해당 async 블록으로 *이동*할 수 있다면, 그 블록이 끝날 때 `tx`가 drop될 것입니다. 13장 [참조 캡처 또는 소유권 이동][capture-or-move]<!-- ignore -->에서 클로저에 `move` 키워드를 사용하는 방법을 배웠고, 16장 [스레드에서 `move` 클로저 사용하기][move-threads]<!-- ignore -->에서 보았듯이, 스레드 작업 시 데이터를 클로저로 이동해야 할 때가 많습니다. async 블록에서도 기본적으로 동일한 원리가 적용되므로, `move` 키워드는 클로저와 마찬가지로 async 블록에도 사용할 수 있습니다.

Listing 17-12에서는 메시지를 보내는 블록을 `async`에서 `async move`로 변경합니다. *이* 버전의 코드를 실행하면, 마지막 메시지가 전송되고 수신된 뒤 프로그램이 정상적으로 종료됩니다.

<Listing number="17-12" caption="Listing 17-11의 코드를 수정하여 모든 작업이 끝나면 올바르게 종료되도록 한 예제" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-12/src/main.rs:with-move}}
```

</Listing>

이 async 채널 역시 다중 생산자 채널이므로, 여러 future에서 메시지를 보내고 싶다면 `tx`에 대해 `clone`을 호출할 수 있습니다. Listing 17-13에서처럼 사용할 수 있습니다.

<Listing number="17-13" caption="여러 프로듀서를 async 블록과 함께 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-13/src/main.rs:here}}
```

</Listing>

먼저, `tx`를 복제해서 첫 번째 async 블록 바깥에 `tx1`을 만듭니다. 그리고 이전에 `tx`를 이동했던 것처럼, `tx1`을 해당 블록으로 이동시킵니다. 그 다음, 원래의 `tx`는 *새로운* async 블록으로 이동시켜, 약간 더 느린 지연으로 메시지를 추가로 보냅니다. 이 새로운 async 블록을 메시지를 받는 async 블록 뒤에 두었지만, 앞에 두어도 상관없습니다. 중요한 것은 future를 *생성하는* 순서가 아니라, future를 *await*하는 순서입니다.

메시지를 보내는 두 async 블록 모두가 `async move` 블록이어야, 두 블록이 끝날 때 각각의 `tx`와 `tx1`이 drop됩니다. 그렇지 않으면 처음에 겪었던 것처럼 무한 루프에 빠지게 됩니다. 마지막으로, 추가된 future를 처리하기 위해 `trpl::join` 대신 `trpl::join3`을 사용합니다.

이제 두 개의 메시지 전송 future에서 온 모든 메시지를 볼 수 있습니다. 그리고 메시지 전송 future들이 메시지를 보낸 뒤 약간씩 다른 지연을 사용하기 때문에, 메시지들도 각각 다른 간격으로 수신됩니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
received 'hi'
received 'more'
received 'from'
received 'the'
received 'messages'
received 'future'
received 'for'
received 'you'
```

좋은 시작이지만, 이 방식은 우리가 사용할 수 있는 future의 수를 제한합니다. 즉, `join`을 사용하면 두 개, `join3`을 사용하면 세 개까지만 가능합니다. 더 많은 future를 다루려면 어떻게 해야 할지 살펴봅시다.

[thread-spawn]: https://doc.rust-lang.org/book/ch16-01-threads.html#creating-a-new-thread-with-spawn
[join-handles]: https://doc.rust-lang.org/book/ch16-01-threads.html#waiting-for-all-threads-to-finish-using-join-handles
[message-passing-threads]: https://doc.rust-lang.org/book/ch16-02-message-passing.html
[if-let]: https://doc.rust-lang.org/book/ch06-03-if-let.html
[capture-or-move]: https://doc.rust-lang.org/book/ch13-01-closures.html#capturing-references-or-moving-ownership
[move-threads]: https://doc.rust-lang.org/book/ch16-01-threads.html#using-move-closures-with-threads