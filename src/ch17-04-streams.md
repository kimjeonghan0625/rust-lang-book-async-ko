## Stream: 순차적인 Future

<!-- Old headings. Do not remove or links may break. -->

<a id="streams"></a>

지금까지 이 장에서는 주로 개별 future에 집중했습니다. 한 가지 큰 예외는 우리가 사용했던 async 채널이었습니다. 앞서 이 장의 [“메시지 전달”][17-02-messages]<!-- ignore --> 섹션에서 async 채널의 수신기를 어떻게 사용했는지 기억해 보세요. async `recv` 메서드는 시간에 따라 일련의 아이템들을 생성합니다. 이것은 _스트림 (stream)_ 이라고 하는 훨씬 더 일반적인 패턴의 한 예입니다.

우리는 13장에서 [이터레이터 트레이트와 `next` 메서드][iterator-trait]<!-- ignore --> 섹션을 살펴보면서 아이템의 시퀀스를 본 적이 있습니다. 하지만 이터레이터와 async 채널 수신기에는 두 가지 차이점이 있습니다. 첫 번째는 시간입니다. 이터레이터는 동기적으로 동작하지만, 채널 수신기는 비동기적으로 동작합니다. 두 번째는 API입니다. `Iterator`를 직접 사용할 때는 동기적인 `next` 메서드를 호출합니다. 반면, 특히 `trpl::Receiver` 스트림에서는 비동기적인 `recv` 메서드를 호출했습니다. 그 외에는 이 API들이 매우 비슷하게 느껴지는데, 이것은 우연이 아닙니다. 스트림은 비동기적인 형태의 반복(iteration)과 같습니다. `trpl::Receiver`가 메시지를 받기 위해 특별히 대기하는 반면, 범용 스트림 API는 훨씬 더 넓은 개념입니다: 이터레이터처럼 다음 아이템을 제공하지만, 비동기적으로 동작합니다.

이터레이터와 스트림의 유사성 덕분에, 러스트에서는 실제로 어떤 이터레이터든 스트림으로 만들 수 있습니다. 이터레이터와 마찬가지로, 스트림에서도 `next` 메서드를 호출한 뒤 그 결과를 await 하여 사용할 수 있습니다. Listing 17-30에서처럼 말이죠.

<Listing number="17-30" caption="이터레이터로부터 스트림을 만들고 그 값을 출력하기" file-name="src/main.rs">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-30/src/main.rs:stream}}
```

</Listing>

우리는 숫자 배열로 시작해서, 이를 이터레이터로 변환한 뒤 `map`을 호출하여 모든 값을 두 배로 만듭니다. 그런 다음 이터레이터를 `trpl::stream_from_iter` 함수를 사용해 스트림으로 변환합니다. 이후, 스트림에 아이템이 도착할 때마다 `while let` 루프로 반복합니다.

하지만 코드를 실행해보면, 컴파일이 되지 않고 `next` 메서드를 사용할 수 없다는 에러가 발생합니다:

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-30
cargo build
copy only the error output
-->

```console
error[E0599]: no method named `next` found for struct `Iter` in the current scope
  --> src/main.rs:10:40
   |
10 |         while let Some(value) = stream.next().await {
   |                                        ^^^^
   |
   = note: the full type name has been written to 'file:///projects/async-await/target/debug/deps/async_await-575db3dd3197d257.long-type-14490787947592691573.txt'
   = note: consider using `--verbose` to print the full type name to the console
   = help: items from traits can only be used if the trait is in scope
help: the following traits which provide `next` are implemented but not in scope; perhaps you want to import one of them
   |
1  + use crate::trpl::StreamExt;
   |
1  + use futures_util::stream::stream::StreamExt;
   |
1  + use std::iter::Iterator;
   |
1  + use std::str::pattern::Searcher;
   |
help: there is a method `try_next` with a similar name
   |
10 |         while let Some(value) = stream.try_next().await {
   |                                        ~~~~~~~~
```

이 출력에서 설명하듯이, 컴파일러 에러의 원인은 `next` 메서드를 사용하려면 올바른 트레이트가 스코프에 있어야 한다는 점입니다. 지금까지의 논의를 바탕으로, 여러분은 아마도 그 트레이트가 `Stream`일 것이라고 예상할 수 있지만, 실제로는 `StreamExt`입니다. _확장(extension)_ 을 의미하는 `Ext`는 러스트 커뮤니티에서 한 트레이트를 다른 트레이트로 확장할 때 자주 사용하는 패턴입니다.

`Stream`과 `StreamExt` 트레이트에 대해서는 이 장의 끝에서 좀 더 자세히 설명하겠지만, 지금은 `Stream` 트레이트가 사실상 `Iterator`와 `Future` 트레이트를 결합한 저수준 인터페이스를 정의한다는 점만 알면 충분합니다. `StreamExt`는 `Stream` 위에 더 높은 수준의 API 집합을 제공하는데, 여기에는 `next` 메서드와 `Iterator` 트레이트가 제공하는 것과 유사한 여러 유틸리티 메서드가 포함되어 있습니다. `Stream`과 `StreamExt`는 아직 러스트 표준 라이브러리의 일부는 아니지만, 대부분의 생태계 크레이트에서는 동일한 정의를 사용합니다.

컴파일러 에러를 해결하려면 Listing 17-31과 같이 `trpl::StreamExt`를 `use`로 가져오면 됩니다.

<Listing number="17-31" caption="이터레이터를 스트림의 기반으로 성공적으로 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-31/src/main.rs:all}}
```

</Listing>

이 모든 요소들을 조합하면, 이제 코드는 우리가 원하는 대로 동작합니다! 게다가 이제 `StreamExt`가 스코프에 있으므로, 이터레이터에서처럼 다양한 유틸리티 메서드들을 모두 사용할 수 있습니다. 예를 들어, Listing 17-32에서는 `filter` 메서드를 사용해 3과 5의 배수만 남기고 나머지를 걸러냅니다.

<Listing number="17-32" caption="Filtering a stream with the `StreamExt::filter` method" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-32/src/main.rs:all}}
```

</Listing>

사실, 지금까지의 예시는 평범한 이터레이터만 사용해도, 비동기 없이 충분히 구현할 수 있기 때문에 그다지 흥미롭지 않습니다. 이제 스트림에서만 할 수 있는 _고유한_ 작업에는 무엇이 있는지 살펴보겠습니다.

### 스트림 구성하기

많은 개념들은 스트림으로 자연스럽게 표현할 수 있습니다. 예를 들어, 큐에 아이템이 순차적으로 들어오는 경우, 전체 데이터를 한 번에 메모리에 올릴 수 없을 때 파일 시스템에서 데이터를 청크 단위로 점진적으로 읽어오는 경우, 또는 네트워크를 통해 데이터가 시간에 따라 도착하는 경우 등이 있습니다. 스트림은 future이기 때문에, 다른 종류의 future와 함께 사용하거나 흥미로운 방식으로 조합할 수 있습니다. 예를 들어, 이벤트를 묶어서 너무 많은 네트워크 호출이 발생하지 않도록 하거나, 오래 걸리는 작업 시퀀스에 타임아웃을 설정하거나, 사용자 인터페이스 이벤트를 *throttle* 해서 불필요한 작업을 줄일 수 있습니다.

먼저, Listing 17-33에서처럼 WebSocket이나 다른 실시간 통신 프로토콜에서 볼 수 있는 데이터 스트림을 대신할 작은 메시지 스트림을 만들어보겠습니다.

<Listing number="17-33" caption="`rx` 수신기를 `ReceiverStream`으로 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-33/src/main.rs:all}}
```

</Listing>

먼저, `get_messages`라는 함수를 만들어 `impl Stream<Item = String>`을 반환하도록 합니다. 이 함수의 구현에서는 async 채널을 생성하고, 영어 알파벳의 처음 10글자를 반복하면서 채널을 통해 전송합니다.

또한 새로운 타입인 `ReceiverStream`을 사용하는데, 이는 `trpl::channel`의 `rx` 수신기를 `next` 메서드를 가진 `Stream`으로 변환해줍니다. 다시 `main` 함수로 돌아와서, `while let` 루프를 사용해 스트림에서 모든 메시지를 출력합니다.

이 코드를 실행하면, 예상한 결과가 정확히 출력됩니다:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Message: 'a'
Message: 'b'
Message: 'c'
Message: 'd'
Message: 'e'
Message: 'f'
Message: 'g'
Message: 'h'
Message: 'i'
Message: 'j'
```

사실, 이 작업은 일반적인 `Receiver` API나 `Iterator` API만으로도 충분히 할 수 있습니다. 하지만 이제 스트림에서만 가능한 기능을 추가해보겠습니다. 바로 스트림의 각 아이템마다 타임아웃을 적용하고, 우리가 내보내는 아이템에 딜레이를 추가하는 것입니다. Listing 17-34에서 그 예시를 볼 수 있습니다.

<Listing number="17-34" caption="스트림의 각 아이템에 시간 제한을 두기 위해 `StreamExt::timeout` 메서드 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-34/src/main.rs:timeout}}
```

</Listing>

먼저, `StreamExt` 트레이트에서 제공하는 `timeout` 메서드를 사용해 스트림에 타임아웃을 추가합니다. 그러면 이제 스트림이 `Result`를 반환하므로, `while let` 루프의 본문도 수정해야 합니다. `Ok`는 메시지가 제때 도착했음을, `Err`는 타임아웃이 발생했음을 의미합니다. 우리는 이 결과를 `match`로 분기하여, 메시지를 정상적으로 받았을 때는 출력하고, 타임아웃이 발생했을 때는 안내 메시지를 출력합니다. 마지막으로, `timeout` 헬퍼가 반환하는 스트림은 폴링하려면 핀이 필요하므로, 타임아웃을 적용한 후에 메시지 스트림을 핀(pin) 처리하는 점에 주의하세요.

하지만 메시지 사이에 딜레이가 없기 때문에, 이 타임아웃은 프로그램의 동작에 아무런 변화를 주지 않습니다. Listing 17-35에서처럼, 이제 전송하는 메시지에 가변적인 딜레이를 추가해보겠습니다.

<Listing number="17-35" caption="`get_messages`를 async 함수로 만들지 않고도 async 딜레이와 함께 `tx`로 메시지를 전송하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-35/src/main.rs:messages}}
```

</Listing>

`get_messages` 함수에서는 `messages` 배열에 `enumerate` 이터레이터 메서드를 사용하여 각 아이템을 전송할 때 그 인덱스와 아이템 자체를 함께 얻습니다. 그리고 실제 환경에서 메시지 스트림에서 볼 수 있는 다양한 지연을 시뮬레이션하기 위해, 인덱스가 짝수인 아이템에는 100밀리초, 홀수인 아이템에는 300밀리초의 딜레이를 적용합니다. 타임아웃이 200밀리초로 설정되어 있으므로, 이로 인해 전체 메시지의 절반에 영향을 주게 됩니다.

`get_messages` 함수에서 메시지 사이에 블로킹 없이 잠시 멈추려면 async를 사용해야 합니다. 하지만 `get_messages` 자체를 async 함수로 만들 수는 없습니다. 그렇게 하면 반환 타입이 `Stream<Item = String>`이 아니라 `Future<Output = Stream<Item = String>>`이 되기 때문입니다. 호출자는 스트림에 접근하기 위해 `get_messages` 자체를 await 해야 합니다. 하지만 기억하세요: 하나의 future 내에서는 모든 일이 순차적으로 일어나고, 동시성은 future _사이_ 에서 발생합니다. `get_messages`를 await 하면, 모든 메시지 전송(각 메시지 사이의 sleep 딜레이 포함)이 스트림을 반환하기 전에 한 번에 처리됩니다. 그 결과, 타임아웃이 무의미해집니다. 스트림 자체에는 지연이 없고, 모든 지연이 스트림이 생성되기 전에 발생하게 됩니다.

따라서, `get_messages`는 스트림을 반환하는 일반 함수로 두고, async `sleep` 호출을 처리할 별도의 태스크를 생성해서 실행합니다.

> 참고: 이렇게 `spawn_task`를 호출할 수 있는 것은 이미 런타임을 설정해두었기 때문입니다. 만약 그렇지 않았다면 패닉이 발생했을 것입니다. 다른 구현체들은 각기 다른 트레이드오프를 선택합니다. 어떤 것은 새로운 런타임을 생성해 패닉을 피하지만 약간의 오버헤드가 생길 수 있고, 어떤 것은 런타임에 대한 참조 없이 태스크를 독립적으로 스폰하는 방법 자체를 제공하지 않을 수도 있습니다. 사용하는 런타임이 어떤 트레이드오프를 선택했는지 반드시 확인하고, 그에 맞게 코드를 작성하세요!

이제 우리의 코드는 훨씬 더 흥미로운 결과를 보여줍니다. 메시지 쌍마다 `Problem: Elapsed(())` 에러가 나타납니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Message: 'a'
Problem: Elapsed(())
Message: 'b'
Message: 'c'
Problem: Elapsed(())
Message: 'd'
Message: 'e'
Problem: Elapsed(())
Message: 'f'
Message: 'g'
Problem: Elapsed(())
Message: 'h'
Message: 'i'
Problem: Elapsed(())
Message: 'j'
```

타임아웃이 있다고 해서 메시지가 결국 도착하지 않는 것은 아닙니다. 우리의 채널이 *무제한(unbounded)* 이기 때문에, 결국 모든 원래 메시지를 다 받게 됩니다. 즉, 메모리에 담을 수 있는 한 많은 메시지를 저장할 수 있습니다. 만약 메시지가 타임아웃 전에 도착하지 않더라도, 스트림 핸들러가 그 상황을 처리하고, 스트림을 다시 폴링할 때 메시지가 도착해 있을 수 있습니다.

필요하다면 다른 종류의 채널이나, 더 일반적으로는 다른 종류의 스트림을 사용해서 다른 동작을 얻을 수도 있습니다. 이제 메시지 스트림과 시간 간격 스트림을 결합하는 예시를 살펴보겠습니다.

### 스트림 합치기

먼저, 또 다른 스트림을 만들어보겠습니다. 이 스트림은 직접 실행하면 1밀리초마다 아이템을 하나씩 내보냅니다. 간단하게, `sleep` 함수를 사용해 일정 시간 후에 메시지를 전송하고, `get_messages`에서 사용했던 것처럼 채널로부터 스트림을 만드는 방식을 결합할 수 있습니다. 차이점은 이번에는 경과한 인터벌의 개수를 반환한다는 점입니다. 따라서 반환 타입은 `impl Stream<Item = u32>`가 되고, 함수 이름은 `get_intervals`로 할 수 있습니다 (Listing 17-36 참고).

<Listing number="17-36" caption="1밀리초마다 카운터를 내보내는 스트림 만들기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-36/src/main.rs:intervals}}
```

</Listing>

우선 태스크 내부에서 `count` 변수를 정의합니다. (태스크 바깥에 정의할 수도 있지만, 변수의 범위를 제한하는 것이 더 명확합니다.) 그런 다음 무한 루프를 만듭니다. 루프의 각 반복마다 1밀리초 동안 비동기로 잠시 멈추고, 카운트를 증가시킨 뒤 채널을 통해 값을 전송합니다. 이 모든 작업은 `spawn_task`로 생성된 태스크 안에서 이루어지므로, 무한 루프를 포함한 전체 작업이 런타임이 종료될 때 함께 정리됩니다.

런타임이 완전히 종료될 때까지 끝나지 않는 이러한 종류의 무한 루프는 async 러스트에서 꽤 흔하게 볼 수 있습니다. 많은 프로그램이 무기한 실행되어야 하기 때문입니다. async를 사용하면 각 반복마다 적어도 하나의 await 지점이 있다면, 이 루프가 다른 작업을 막지 않습니다.

이제 다시 main 함수의 async 블록으로 돌아와서, Listing 17-37에서처럼 `messages` 스트림과 `intervals` 스트림을 합치는 시도를 할 수 있습니다.

<Listing number="17-37" caption="`messages` 스트림과 `intervals` 스트림을 합치려는 시도" file-name="src/main.rs">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-37/src/main.rs:main}}
```

</Listing>

먼저 `get_intervals`를 호출합니다. 그런 다음 `merge` 메서드를 사용해 `messages` 스트림과 `intervals` 스트림을 합칩니다. `merge`는 여러 스트림을 하나로 결합하여, 각 소스 스트림에서 아이템이 준비되는 즉시 순서에 상관없이 아이템을 생성하는 스트림을 만듭니다. 마지막으로, 이제는 `messages` 대신 결합된 스트림을 반복(loop)합니다.

이 시점에서는, `messages`와 `intervals` 모두 단일 `merged` 스트림에 결합되므로 핀(pin) 처리나 가변(mut) 처리가 필요하지 않습니다. 하지만 이 `merge` 호출은 컴파일되지 않습니다! (`while let` 루프의 `next` 호출도 마찬가지지만, 이 부분은 나중에 다루겠습니다.) 그 이유는 두 스트림의 타입이 다르기 때문입니다. `messages` 스트림의 타입은 `Timeout<impl Stream<Item = String>>`이고, 여기서 `Timeout`은 `timeout` 호출에 대해 `Stream`을 구현하는 타입입니다. 반면, `intervals` 스트림의 타입은 `impl Stream<Item = u32>`입니다. 이 두 스트림을 합치려면, 둘 중 하나의 타입을 맞춰야 합니다. 이미 우리가 원하는 기본 포맷이고 타임아웃 에러도 처리해야 하는 `messages` 쪽은 그대로 두고, `intervals` 스트림을 변환하는 것이 좋습니다 (Listing 17-38 참고).

<!-- We cannot directly test this one, because it never stops. -->

<Listing number="17-38" caption="`intervals` 스트림의 타입을 `messages` 스트림의 타입과 맞추기" file-name="src/main.rs">

```rust,ignore
{{#rustdoc_include ../listings/ch17-async-await/listing-17-38/src/main.rs:main}}
```

</Listing>

먼저, `map` 헬퍼 메서드를 사용해 `intervals`를 문자열로 변환할 수 있습니다. 두 번째로, `messages`에서 사용하는 `Timeout` 타입과 맞춰야 합니다. 하지만 `intervals`에는 실제로 타임아웃이 필요하지 않으므로, 우리가 사용하는 다른 딜레이보다 충분히 긴 타임아웃을 그냥 만들어주면 됩니다. 여기서는 `Duration::from_secs(10)`을 사용해 10초짜리 타임아웃을 만듭니다. 마지막으로, `while let` 루프에서 `next`를 반복 호출할 수 있도록 `stream`을 가변(mut) 변수로 만들고, 안전하게 사용할 수 있도록 핀(pin) 처리해야 합니다. 이렇게 하면 _거의_ 원하는 결과에 도달합니다. 모든 타입이 맞춰집니다. 하지만 실제로 실행해보면 두 가지 문제가 있습니다. 첫째, 프로그램이 절대 종료되지 않습니다! <span class="keystroke">ctrl-c</span>로 직접 중단해야 합니다. 둘째, 알파벳 메시지들이 모든 인터벌 카운터 메시지 사이에 묻혀버립니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the tasks running differently rather than
changes in the compiler -->

```text
--snip--
Interval: 38
Interval: 39
Interval: 40
Message: 'a'
Interval: 41
Interval: 42
Interval: 43
--snip--
```
Listing 17-39는 이 마지막 두 가지 문제를 해결하는 한 가지 방법을 보여줍니다.

<Listing number="17-39" caption="`throttle`와 `take`를 사용해 합쳐진 스트림을 관리하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-39/src/main.rs:throttle}}
```

</Listing>

먼저, `intervals` 스트림에 `throttle` 메서드를 사용하여 `messages` 스트림이 압도당하지 않도록 합니다. *Throttling*은 함수가 호출되는 빈도, 즉 여기서는 스트림이 폴링되는 빈도를 제한하는 방법입니다. 메시지가 도착하는 간격과 비슷하게 100밀리초마다 한 번씩이면 충분합니다.

스트림에서 받아들일 아이템의 개수를 제한하려면, `merged` 스트림에 `take` 메서드를 적용합니다. 이는 한쪽 스트림이 아니라 최종 출력 전체를 제한하기 위함입니다.

이제 프로그램을 실행하면, 스트림에서 20개의 아이템을 가져온 뒤 종료되고, 인터벌 메시지가 메시지 스트림을 압도하지 않습니다. 또한 `Interval: 100`이나 `Interval: 200` 같은 값이 아니라, `Interval: 1`, `Interval: 2`처럼 순차적으로 출력됩니다. 비록 소스 스트림이 1밀리초마다 이벤트를 생성할 수 있지만, `throttle` 호출이 원래 스트림을 감싸서 지정한 주기마다만 폴링하도록 만들기 때문입니다. 즉, 무시되는 인터벌 메시지가 쌓이는 것이 아니라, 애초에 그런 메시지가 생성되지 않습니다! 이것이 러스트의 future가 가진 *지연(laziness)* 특성으로, 우리가 원하는 성능 특성을 직접 선택할 수 있게 해줍니다.

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
Interval: 1
Message: 'a'
Interval: 2
Interval: 3
Problem: Elapsed(())
Interval: 4
Message: 'b'
Interval: 5
Message: 'c'
Interval: 6
Interval: 7
Problem: Elapsed(())
Interval: 8
Message: 'd'
Interval: 9
Message: 'e'
Interval: 10
Interval: 11
Problem: Elapsed(())
Interval: 12
```

마지막으로 처리해야 할 것이 하나 남았습니다. 바로 에러입니다! 이 두 채널 기반 스트림 모두에서, `send` 호출은 반대편에서 채널이 닫힐 때 실패할 수 있습니다. 이는 스트림을 구성하는 future들이 런타임에서 어떻게 실행되는지에 따라 달라집니다. 지금까지는 단순히 `unwrap`을 호출해서 이 가능성을 무시했지만, 실제 애플리케이션에서는 최소한 더 이상 메시지를 보내지 않도록 루프를 종료하는 등 명시적으로 에러를 처리해야 합니다. Listing 17-40은 간단한 에러 처리 전략을 보여줍니다. 문제가 발생하면 에러를 출력하고, 루프에서 `break`로 빠져나옵니다.

<Listing number="17-40" caption="에러를 처리하고 루프를 종료하기">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-40/src/main.rs:errors}}
```

</Listing>

항상 그렇듯이, 메시지 전송 에러를 처리하는 올바른 방법은 상황에 따라 다릅니다. 중요한 것은 반드시 자신만의 전략을 세워두는 것입니다.

이제 실제 예시를 통해 async를 충분히 살펴보았으니, 잠시 한 걸음 물러나서 러스트에서 async가 동작하는 방식의 핵심인 `Future`, `Stream`, 그리고 기타 주요 트레이트들의 세부 사항을 좀 더 깊이 파고들어 보겠습니다.

[17-02-messages]: https://doc.rust-lang.org/book/ch17-02-concurrency-with-async.html#message-passing
[iterator-trait]: https://doc.rust-lang.org/book/ch13-02-iterators.html#the-iterator-trait-and-the-next-method