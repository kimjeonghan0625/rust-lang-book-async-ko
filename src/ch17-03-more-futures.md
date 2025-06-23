## 여러 Future 다루기

이전 섹션에서 두 개의 future에서 세 개의 future로 바꿨을 때, `join`에서 `join3`으로 바꿔야 했습니다. future의 개수가 바뀔 때마다 매번 다른 함수를 호출해야 한다면 꽤 번거로울 것입니다. 다행히도, 임의 개수의 인자를 받을 수 있는 `join`의 매크로 형태가 있습니다. 이 매크로는 future들을 직접 await 해주기도 합니다. 따라서 Listing 17-13의 코드를 `join3` 대신 `join!`을 사용하도록 Listing 17-14와 같이 다시 쓸 수 있습니다.

<Listing number="17-14" caption="여러 future를 기다리기 위해 `join!` 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-14/src/main.rs:here}}
```

</Listing>

이 방식은 `join`, `join3`, `join4` 등으로 계속 바꿔가며 사용하는 것보다 확실히 개선된 방법입니다! 하지만 이 매크로 형태 역시 future의 개수를 미리 알고 있을 때만 동작합니다. 실제 Rust 코드에서는 future들을 컬렉션에 담아두고, 그 중 일부 또는 전부가 완료될 때까지 기다리는 패턴이 흔하게 사용됩니다.

컬렉션에 있는 모든 future를 확인하려면, *모두*를 순회(iterate)하면서 join해야 합니다. `trpl::join_all` 함수는 [이터레이터 트레잇과 `next` 메서드][iterator-trait]<!-- ignore --> (13장)에서 배운 것처럼, `Iterator` 트레잇을 구현하는 어떤 타입도 받을 수 있으므로 딱 맞는 도구처럼 보입니다. 이제 future들을 벡터에 담고, `join!` 대신 `join_all`을 사용하는 방법을 Listing 17-15에서 살펴보겠습니다.

<Listing number="17-15" caption="익명 future들을 벡터에 저장하고 `join_all` 호출하기">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-15/src/main.rs:here}}
```

</Listing>

안타깝게도, 이 코드는 컴파일되지 않습니다. 대신, 다음과 같은 에러가 발생합니다:

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-15/
cargo build
copy just the compiler error
-->

```text
error[E0308]: mismatched types
  --> src/main.rs:45:37
   |
10 |         let tx1_fut = async move {
   |                       ---------- the expected `async` block
...
24 |         let rx_fut = async {
   |                      ----- the found `async` block
...
45 |         let futures = vec![tx1_fut, rx_fut, tx_fut];
   |                                     ^^^^^^ expected `async` block, found a different `async` block
   |
   = note: expected `async` block `{async block@src/main.rs:10:23: 10:33}`
              found `async` block `{async block@src/main.rs:24:22: 24:27}`
   = note: no two async blocks, even if identical, have the same type
   = help: consider pinning your async block and casting it to a trait object
```

이 점은 다소 놀라울 수 있습니다. 결국, async 블록 중 어느 것도 값을 반환하지 않으므로 각각은 `Future<Output = ()>`를 생성합니다. 하지만 `Future`는 트레잇이라는 점을 기억하세요. 컴파일러는 각 async 블록마다 고유한 열거형(enum)을 만듭니다. 서로 다른 구조체(struct)를 `Vec`에 넣을 수 없는 것처럼, 컴파일러가 생성한 서로 다른 열거형도 마찬가지로 `Vec`에 함께 담을 수 없습니다.

이 문제를 해결하려면, 12장에서 [*run* 함수에서 에러 반환하기][dyn]<!-- ignore -->에서 했던 것처럼 *트레잇 객체*를 사용해야 합니다. (트레잇 객체에 대해서는 18장에서 자세히 다룹니다.) 트레잇 객체를 사용하면, 이러한 익명 future 각각을 동일한 타입으로 다룰 수 있습니다. 왜냐하면 이들은 모두 `Future` 트레잇을 구현하기 때문입니다.

> 참고: 8장의 [여러 값을 저장하기 위해 열거형 사용하기][enum-alt]<!-- ignore -->에서, `Vec`에 여러 타입을 담는 또 다른 방법으로 각 타입을 나타내는 열거형(enum)을 사용하는 방법을 다뤘습니다. 하지만 여기서는 그렇게 할 수 없습니다. 첫째, 이 경우에는 타입들이 익명이라 이름을 붙일 방법이 없습니다. 둘째, 우리가 벡터와 `join_all`을 사용하려는 이유는, 오직 *동일한 출력 타입*을 가지는 동적인 future 컬렉션을 다루고 싶기 때문입니다.

먼저, Listing 17-16에서처럼 `vec!`에 담긴 각 future를 `Box::new`로 감쌉니다.

<Listing number="17-16" caption="`Box::new`을 사용해 `Vec`에 담긴 future들의 타입을 맞추기" file-name="src/main.rs">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-16/src/main.rs:here}}
```

</Listing>

안타깝게도, 이 코드는 여전히 컴파일되지 않습니다. 실제로, 두 번째와 세 번째 `Box::new` 호출에서 앞서 봤던 것과 동일한 기본 에러가 발생하며, `Unpin` 트레잇과 관련된 새로운 에러도 나타납니다. `Unpin` 에러는 잠시 뒤에 다루겠습니다. 먼저, `Box::new` 호출에서 발생하는 타입 에러를 해결하기 위해 `futures` 변수의 타입을 명시적으로 지정해줍시다 (Listing 17-17 참고).

<Listing number="17-17" caption="명시적인 타입 선언으로 나머지 타입 불일치 에러 해결하기" file-name="src/main.rs">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-17/src/main.rs:here}}
```

</Listing>

이 타입 선언은 조금 복잡하니, 단계별로 살펴보겠습니다.

1. 가장 안쪽 타입은 future 자체입니다. future의 출력이 단위 타입 `()`임을 명시적으로 `Future<Output = ()>`로 표시합니다.
2. 그런 다음, 트레잇에 `dyn`을 붙여 동적임을 나타냅니다.
3. 전체 트레잇 참조를 `Box`로 감쌉니다.
4. 마지막으로, `futures`가 이러한 항목들을 담는 `Vec`임을 명확히 명시합니다.

이것만으로도 큰 차이가 생겼습니다. 이제 컴파일러를 실행하면 `Unpin`과 관련된 에러만 남게 됩니다. 비슷한 내용의 에러가 세 개 나오지만, 모두 거의 동일합니다.

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-17
cargo build
# copy *only* the errors
# fix the paths
-->

```text
error[E0277]: `dyn Future<Output = ()>` cannot be unpinned
   --> src/main.rs:49:24
    |
49  |         trpl::join_all(futures).await;
    |         -------------- ^^^^^^^ the trait `Unpin` is not implemented for `dyn Future<Output = ()>`
    |         |
    |         required by a bound introduced by this call
    |
    = note: consider using the `pin!` macro
            consider using `Box::pin` if you need to access the pinned value outside of the current scope
    = note: required for `Box<dyn Future<Output = ()>>` to implement `Future`
note: required by a bound in `join_all`
   --> file:///home/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/futures-util-0.3.30/src/future/join_all.rs:105:14
    |
102 | pub fn join_all<I>(iter: I) -> JoinAll<I::Item>
    |        -------- required by a bound in this function
...
105 |     I::Item: Future,
    |              ^^^^^^ required by this bound in `join_all`

error[E0277]: `dyn Future<Output = ()>` cannot be unpinned
  --> src/main.rs:49:9
   |
49 |         trpl::join_all(futures).await;
   |         ^^^^^^^^^^^^^^^^^^^^^^^ the trait `Unpin` is not implemented for `dyn Future<Output = ()>`
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
   = note: required for `Box<dyn Future<Output = ()>>` to implement `Future`
note: required by a bound in `futures_util::future::join_all::JoinAll`
  --> file:///home/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/futures-util-0.3.30/src/future/join_all.rs:29:8
   |
27 | pub struct JoinAll<F>
   |            ------- required by a bound in this struct
28 | where
29 |     F: Future,
   |        ^^^^^^ required by this bound in `JoinAll`

error[E0277]: `dyn Future<Output = ()>` cannot be unpinned
  --> src/main.rs:49:33
   |
49 |         trpl::join_all(futures).await;
   |                                 ^^^^^ the trait `Unpin` is not implemented for `dyn Future<Output = ()>`
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
   = note: required for `Box<dyn Future<Output = ()>>` to implement `Future`
note: required by a bound in `futures_util::future::join_all::JoinAll`
  --> file:///home/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/futures-util-0.3.30/src/future/join_all.rs:29:8
   |
27 | pub struct JoinAll<F>
   |            ------- required by a bound in this struct
28 | where
29 |     F: Future,
   |        ^^^^^^ required by this bound in `JoinAll`

For more information about this error, try `rustc --explain E0277`.
error: could not compile `async_await` (bin "async_await") due to 3 previous errors
```

소화하기에 *꽤 많은* 내용이었으니, 하나씩 분해해서 살펴봅시다. 메시지의 첫 부분은 첫 번째 async 블록(`src/main.rs:8:23: 20:10`)이 `Unpin` 트레잇을 구현하지 않았다고 알려주며, 이를 해결하기 위해 `pin!`이나 `Box::pin`을 사용하라고 제안합니다. 이 장의 뒷부분에서 `Pin`과 `Unpin`에 대해 좀 더 자세히 다룰 예정입니다. 하지만 지금은 컴파일러의 조언을 따라 문제를 해결해 봅시다. Listing 17-18에서는 먼저 `std::pin`에서 `Pin`을 임포트합니다. 그 다음, `futures`의 타입 어노테이션을 각 `Box`를 감싸는 `Pin`으로 수정합니다. 마지막으로, 실제 future들을 pinning하기 위해 `Box::pin`을 사용합니다.

<Listing number="17-18" caption="`Pin`과 `Box::pin`을 사용해 `Vec`의 타입을 맞추기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-18/src/main.rs:here}}
```

</Listing>

이제 이 코드를 컴파일하고 실행하면, 드디어 기대했던 출력 결과를 얻을 수 있습니다:

<!-- Not extracting output because changes to this output aren't significant;
the changes are likely to be due to the threads running differently rather than
changes in the compiler -->

```text
received 'hi'
received 'more'
received 'from'
received 'messages'
received 'the'
received 'for'
received 'future'
received 'you'
```
휴!

여기서 더 살펴볼 내용이 조금 있습니다. 우선, `Pin<Box<T>>`를 사용하면 future들을 힙에 `Box`로 할당하게 되어 약간의 오버헤드가 발생합니다. 사실 우리는 타입을 맞추기 위해서만 힙 할당을 하고 있을 뿐, 실제로는 힙 할당이 꼭 필요한 것은 아닙니다. 어차피 이 future들은 해당 함수 내부에서만 사용되기 때문입니다. 앞서 언급했듯이, `Pin` 자체도 래퍼 타입이므로, `Box`를 사용했던 원래 이유였던 `Vec`에 단일 타입을 담는 이점을 힙 할당 없이도 얻을 수 있습니다. 각 future에 대해 `std::pin::pin` 매크로를 사용해 직접 `Pin`을 적용할 수 있습니다.

하지만 여전히 핀된 참조의 타입을 명시적으로 지정해야 합니다. 그렇지 않으면 Rust는 이 값들을 우리가 `Vec`에 필요로 하는 동적 트레잇 객체로 해석하지 못합니다. 따라서 `std::pin`에서 `pin`을 임포트 목록에 추가해야 합니다. 그런 다음 future를 정의할 때마다 `pin!`을 사용하고, Listing 17-19처럼 `futures`를 동적 future 타입에 대한 핀된 가변 참조들의 `Vec`으로 정의할 수 있습니다.

<Listing number="17-19" caption="불필요한 힙 할당을 피하기 위해 `pin!` 매크로로 `Pin`을 직접 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-19/src/main.rs:here}}
```

</Listing>

여기까지는 서로 다른 `Output` 타입을 가질 수 있다는 사실을 무시하고 진행해 왔습니다. 예를 들어, Listing 17-20에서 `a`의 익명 future는 `Future<Output = u32>`를 구현하고, `b`의 익명 future는 `Future<Output = &str>`를 구현하며, `c`의 익명 future는 `Future<Output = bool>`을 구현합니다.

<Listing number="17-20" caption="서로 다른 타입을 가진 세 개의 future" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-20/src/main.rs:here}}
```

</Listing>

`trpl::join!`을 사용하면 여러 future 타입을 한 번에 await할 수 있습니다. 이 매크로는 다양한 타입의 future를 받아서 해당 타입들의 튜플을 만들어줍니다. 반면, `trpl::join_all`은 사용할 수 없습니다. 이 함수는 전달된 모든 future가 동일한 타입이어야 하기 때문입니다. 바로 이 타입 에러 때문에 우리가 `Pin`을 사용하게 된 것이죠!

이것은 근본적인 트레이드오프입니다. 만약 future들의 타입이 모두 같다면, 개수가 동적인 경우에는 `join_all`을 사용할 수 있습니다. 반대로 future들의 타입이 다르다면, 개수가 정해진 경우에만 `join` 함수나 `join!` 매크로를 사용할 수 있습니다. 이는 Rust에서 다른 타입을 다룰 때와 동일한 상황입니다. future는 특별한 것이 아니며, 단지 다루기 편한 문법이 추가된 것뿐입니다. 오히려 이것이 좋은 점이기도 합니다.

### Future 경주시키기

`join` 계열의 함수와 매크로로 future들을 *조인*할 때는, *모든* future가 완료될 때까지 기다려야만 다음 단계로 넘어갈 수 있습니다. 하지만 때로는 여러 future 중 *일부*만 완료되면 바로 다음 단계로 넘어가고 싶을 때도 있습니다. 이는 마치 한 future와 다른 future를 경주시키는 것과 비슷합니다.

Listing 17-21에서는 다시 한 번 `trpl::race`를 사용해 두 개의 future, `slow`와 `fast`를 서로 경주시키는 예제를 보여줍니다.

<Listing number="17-21" caption="`race`를 사용해 먼저 끝나는 future의 결과 얻기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-21/src/main.rs:here}}
```

</Listing>

각 future는 실행을 시작할 때 메시지를 출력하고, `sleep`을 호출해 await 하면서 잠시 멈췄다가, 완료되면 다시 메시지를 출력합니다. 그런 다음 `slow`와 `fast`를 모두 `trpl::race`에 전달하고, 둘 중 하나가 끝날 때까지 기다립니다. (여기서 결과는 그리 놀랍지 않습니다. `fast`가 이깁니다.) [*첫 번째 async 프로그램*][async-program]<!-- ignore -->에서 `race`를 사용했을 때와 달리, 여기서는 반환된 `Either` 인스턴스를 그냥 무시합니다. 왜냐하면 모든 흥미로운 동작이 async 블록 내부에서 일어나기 때문입니다.

만약 `race`에 전달하는 인자의 순서를 바꾼다면, “started” 메시지의 출력 순서도 바뀐다는 점에 주목하세요. 비록 항상 `fast` future가 먼저 완료되더라도 말이죠. 이는 이 `race` 함수의 구현이 공정하지 않기 때문입니다. 인자로 전달된 future들을 항상 그 순서대로 실행합니다. 다른 구현체들은 *공정*해서 어떤 future를 먼저 poll할지 무작위로 선택하기도 합니다. 우리가 사용하는 `race` 구현이 공정하든 아니든, *하나*의 future는 본문에서 첫 번째 `await`까지 실행된 뒤에야 다른 태스크가 시작될 수 있습니다.

[*첫 번째 async 프로그램*][async-program]<!-- ignore -->에서 설명했듯이, await 지점마다 Rust는 런타임에게 태스크를 일시 중단하고, 만약 await 중인 future가 준비되지 않았다면 다른 태스크로 전환할 기회를 줍니다. 그 반대도 마찬가지입니다. Rust는 오직 await 지점에서만 async 블록을 일시 중단하고 런타임에 제어권을 넘깁니다. await 지점 사이의 모든 코드는 동기적으로 실행됩니다.

즉, await 지점 없이 async 블록에서 많은 작업을 수행하면, 해당 future가 다른 future들의 진행을 막게 됩니다. 이런 상황을 다른 future가 _굶주린다(starving)_ 고 표현하기도 합니다. 경우에 따라서는 큰 문제가 아닐 수도 있습니다. 하지만 만약 비용이 많이 드는 초기화 작업이나 오래 걸리는 작업을 하거나, 특정 작업을 계속해서 수행하는 future가 있다면, 언제 어디서 런타임에 제어권을 넘길지 고민해야 합니다.

마찬가지로, 오래 걸리는 블로킹 작업이 있다면, async는 프로그램의 여러 부분이 서로 관계를 맺을 수 있는 방법을 제공하는 유용한 도구가 될 수 있습니다.

그렇다면 *어떻게* 이런 경우에 런타임에 제어권을 넘길 수 있을까요?

<!-- Old headings. Do not remove or links may break. -->

<a id="yielding"></a>

### 런타임에 제어권 넘기기

오래 걸리는 작업을 시뮬레이션해 봅시다. Listing 17-22에서는 `slow` 함수를 소개합니다.

<Listing number="17-22" caption="느린 연산을 시뮬레이션하기 위해 `thread::sleep` 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-22/src/main.rs:slow}}
```

</Listing>

이 코드는 `trpl::sleep` 대신 `std::thread::sleep`을 사용하므로, `slow`를 호출하면 현재 스레드가 지정한 밀리초(ms) 동안 블록됩니다. 우리는 `slow`를 실제로 오래 걸리고 블로킹되는 연산을 대신하는 용도로 사용할 수 있습니다.

Listing 17-23에서는 두 개의 future에서 이러한 CPU 바운드 작업을 에뮬레이션하기 위해 `slow`를 사용합니다.

<Listing number="17-23" caption="느린 연산을 시뮬레이션하기 위해 `thread::sleep` 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-23/src/main.rs:slow-futures}}
```

</Listing>

먼저, 각 future는 여러 번의 느린 연산을 *마친 뒤에만* 런타임에 제어권을 넘깁니다. 이 코드를 실행하면 다음과 같은 출력이 나타납니다:

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-23/
cargo run
copy just the output
-->

```text
'a' started.
'a' ran for 30ms
'a' ran for 10ms
'a' ran for 20ms
'b' started.
'b' ran for 75ms
'b' ran for 10ms
'b' ran for 15ms
'b' ran for 350ms
'a' finished.
```

앞서 살펴본 예제와 마찬가지로, `race`는 여전히 `a`가 끝나는 즉시 완료됩니다. 하지만 두 future 사이에 작업이 *섞여서* 실행되지는 않습니다. `a` future는 `trpl::sleep` 호출이 await될 때까지 모든 작업을 마친 뒤, 그제서야 `b` future가 자신의 `trpl::sleep` 호출이 await될 때까지 모든 작업을 수행하고, 마지막으로 `a` future가 완료됩니다. 두 future가 느린 작업 사이에 모두 번갈아가며 진행하려면, await 지점이 필요합니다. 그래야 런타임에 제어권을 넘길 수 있습니다. 즉, await할 수 있는 무언가가 필요하다는 뜻입니다!

이런 제어권 넘김이 Listing 17-23에서 이미 일어나고 있음을 볼 수 있습니다. 만약 `a` future의 마지막에 있는 `trpl::sleep`을 제거한다면, `a`가 완료될 때까지 `b` future는 *전혀* 실행되지 않을 것입니다. 이제 Listing 17-24에서처럼, 연산이 번갈아가며 진행될 수 있도록 `sleep` 함수를 활용해 보겠습니다.

<Listing number="17-24" caption="작업이 번갈아가며 진행되도록 `sleep` 사용하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-24/src/main.rs:here}}
```

</Listing>

Listing 17-24에서는 각 `slow` 호출 사이에 await 지점이 있는 `trpl::sleep` 호출을 추가했습니다. 이제 두 future의 작업이 번갈아가며(interleaved) 실행됩니다:

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-24
cargo run
copy just the output
-->

```text
'a' started.
'a' ran for 30ms
'b' started.
'b' ran for 75ms
'a' ran for 10ms
'b' ran for 10ms
'a' ran for 20ms
'b' ran for 15ms
'a' finished.
```

`a` future는 여전히 제어권을 `b`에게 넘기기 전에 잠시 실행을 계속합니다. 왜냐하면 처음에 `trpl::sleep`을 호출하기 전에 `slow`를 먼저 호출하기 때문입니다. 하지만 그 이후에는 각 future가 await 지점에 도달할 때마다 서로 번갈아가며 실행됩니다. 이 예제에서는 매번 `slow`를 호출한 뒤에 작업을 나눴지만, 실제로는 우리가 원하는 방식대로 작업을 쪼갤 수 있습니다.

여기서 정말로 *sleep*을 하고 싶은 것은 아닙니다. 가능한 한 빠르게 진행하고 싶을 뿐이고, 단지 런타임에 제어권만 넘기면 됩니다. 이를 위해 `yield_now` 함수를 직접 사용할 수 있습니다. Listing 17-25에서는 모든 `sleep` 호출을 `yield_now`로 교체합니다.

<Listing number="17-25" caption="`yield_now`를 사용해 작업이 번갈아가며 진행되도록 하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-25/src/main.rs:yields}}
```

</Listing>

이 코드는 실제 의도를 더 명확하게 보여줄 뿐만 아니라, `sleep`을 사용하는 것보다 훨씬 더 빠를 수 있습니다. 왜냐하면 `sleep`에서 사용하는 타이머와 같은 기능들은 보통 얼마나 세밀하게 동작할 수 있는지에 한계가 있기 때문입니다. 예를 들어, 우리가 사용하는 `sleep`은 1나노초짜리 `Duration`을 전달해도 항상 최소 1밀리초는 잠들게 됩니다. 다시 말해, 현대 컴퓨터는 *정말* 빠르기 때문에 1밀리초 동안 엄청나게 많은 일을 할 수 있습니다!

이 차이를 직접 확인해보고 싶다면, Listing 17-26과 같은 간단한 벤치마크를 만들어 볼 수 있습니다. (이 방법이 아주 엄밀한 성능 테스트는 아니지만, 여기서 차이를 보여주기에는 충분합니다.)

<Listing number="17-26" caption="`sleep`과 `yield_now`의 성능 비교하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-26/src/main.rs:here}}
```

</Listing>

여기서는 상태 출력은 모두 생략하고, `trpl::sleep`에 1나노초짜리 `Duration`을 전달하여 각 future가 서로 전환 없이 독립적으로 실행되도록 합니다. 그런 다음 1,000번 반복해서, `trpl::sleep`을 사용하는 future와 `trpl::yield_now`를 사용하는 future가 각각 얼마나 걸리는지 비교해 봅니다.

`yield_now` 버전이 *훨씬* 더 빠릅니다!

이는 프로그램에서 어떤 작업을 하느냐에 따라, async가 계산 집약적인 작업에도 유용할 수 있음을 의미합니다. async는 프로그램의 여러 부분 간 관계를 구조화하는 데 유용한 도구를 제공하기 때문입니다. 이것은 *협력적 멀티태스킹*의 한 형태로, 각 future가 await 지점을 통해 언제 제어권을 넘길지 스스로 결정할 수 있습니다. 따라서 각 future는 너무 오래 블로킹하지 않도록 책임도 함께 집니다. 일부 Rust 기반 임베디드 운영체제에서는 이것이 *유일한* 멀티태스킹 방식이기도 합니다!

실제 코드에서는 물론 모든 줄마다 함수 호출과 await 지점을 번갈아가며 사용하지는 않습니다. 이런 방식으로 제어권을 넘기는 것은 비교적 비용이 적지만, 완전히 무료는 아닙니다. 많은 경우, 계산 집약적인 작업을 너무 잘게 쪼개면 오히려 성능이 크게 저하될 수 있으므로, *전체적인* 성능을 위해서는 잠깐 블로킹하는 것이 더 나을 때도 있습니다. 항상 실제 코드의 성능 병목이 어디인지 측정해 보세요. 하지만 만약 동시에 실행되길 기대했던 작업이 직렬로 처리되는 현상이 자주 보인다면, 이러한 동작 원리를 꼭 기억해 두세요!

### 직접 Async 추상화 만들기

Future들을 조합해서 새로운 패턴을 만들 수도 있습니다. 예를 들어, 이미 가지고 있는 async 빌딩 블록들로 `timeout` 함수를 만들어볼 수 있습니다. 완성된 결과물 역시 또 다른 빌딩 블록이 되어, 더 많은 async 추상화를 만드는 데 사용할 수 있습니다.

Listing 17-27은 느린 future와 함께 이 `timeout`이 어떻게 동작할지 보여줍니다.

<Listing number="17-27" caption="상상 속의 `timeout`을 사용해 느린 연산을 제한 시간 내에 실행하기" file-name="src/main.rs">

```rust,ignore
{{#rustdoc_include ../listings/ch17-async-await/listing-17-27/src/main.rs:here}}
```

</Listing>

이제 직접 구현해 봅시다! 먼저, `timeout`의 API를 어떻게 설계할지 생각해 보겠습니다.

- 직접 await할 수 있도록, 이 함수 자체도 async 함수여야 합니다.
- 첫 번째 파라미터는 실행할 future가 되어야 합니다. 어떤 future든 동작할 수 있도록 제네릭으로 만들 수 있습니다.
- 두 번째 파라미터는 대기할 최대 시간입니다. `Duration` 타입을 사용하면 `trpl::sleep`에 그대로 넘기기 쉽습니다.
- 반환값은 `Result`여야 합니다. future가 정상적으로 완료되면, 그 결과 값을 담은 `Ok`를 반환합니다. 만약 타임아웃이 먼저 발생하면, 대기한 시간을 담은 `Err`를 반환합니다.

Listing 17-28에서 이 선언을 보여줍니다.

<!-- This is not tested because it intentionally does not compile. -->

<Listing number="17-28" caption="`timeout`의 시그니처 정의하기" file-name="src/main.rs">

```rust,ignore
{{#rustdoc_include ../listings/ch17-async-await/listing-17-28/src/main.rs:declaration}}
```

</Listing>

이제 타입에 대한 목표는 충족했으니, *동작*에 대해 생각해 봅시다. 우리가 원하는 것은 전달받은 future와 지정한 시간(duration)을 경주시키는 것입니다. duration으로부터 타이머 future를 만들기 위해 `trpl::sleep`을 사용할 수 있고, 이 타이머와 호출자가 넘긴 future를 함께 실행하기 위해 `trpl::race`를 사용할 수 있습니다.

또한, `race`는 공정하지 않으며, 인자를 전달된 순서대로 poll한다는 점도 알고 있습니다. 따라서 `future_to_try`를 먼저 `race`에 전달해야, `max_time`이 아주 짧은 경우에도 이 future가 완료될 기회를 얻을 수 있습니다. 만약 `future_to_try`가 먼저 끝나면, `race`는 해당 future의 출력값을 담은 `Left`를 반환합니다. 반대로 타이머가 먼저 끝나면, `race`는 타이머의 출력값인 `()`을 담은 `Right`를 반환합니다.

Listing 17-29에서는 `trpl::race`를 await한 결과를 match로 분기합니다.

<Listing number="17-29" caption="`race`와 `sleep`으로 `timeout` 정의하기" file-name="src/main.rs">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-29/src/main.rs:implementation}}
```

</Listing>

`future_to_try`가 성공해서 `Left(output)`을 받으면 `Ok(output)`을 반환합니다. 반대로 sleep 타이머가 먼저 끝나서 `Right(())`를 받으면, `()`는 `_`로 무시하고 대신 `Err(max_time)`을 반환합니다.

이렇게 하면 두 개의 async 헬퍼로 동작하는 `timeout`을 완성할 수 있습니다. 코드를 실행하면, 타임아웃이 발생했을 때 실패 메시지가 출력됩니다:

```text
Failed after 2 seconds
```

future는 다른 future와 조합할 수 있기 때문에, 더 작은 async 빌딩 블록들을 이용해 매우 강력한 도구를 만들 수 있습니다. 예를 들어, 이와 같은 접근법을 사용해 타임아웃과 재시도를 결합하고, 이를 네트워크 호출(이 장의 앞부분에서 다룬 예시 중 하나)과 같은 작업에 적용할 수도 있습니다.

실제로는 대부분 `async`와 `await`를 직접 사용하게 되며, 그 다음으로는 `join`, `join_all`, `race` 등과 같은 함수나 매크로를 사용하게 됩니다. 이러한 API와 함께 future를 사용하려면 가끔씩만 `pin`을 사용할 필요가 있습니다.

이제 여러 future를 동시에 다루는 다양한 방법을 살펴보았습니다. 다음으로는, 시간에 따라 여러 future를 순차적으로 다루는 *스트림(stream)* 에 대해 알아보겠습니다. 그 전에 고려해볼 만한 몇 가지 사항이 있습니다:

- 우리는 `join_all`과 함께 `Vec`을 사용해, 어떤 그룹에 속한 모든 future가 완료될 때까지 기다렸습니다. 만약 `Vec`을 사용해 future 그룹을 *순차적으로* 처리하려면 어떻게 해야 할까요? 그렇게 했을 때의 트레이드오프는 무엇일까요?

- `futures` 크레이트의 `futures::stream::FuturesUnordered` 타입을 살펴보세요. 이것을 `Vec` 대신 사용하면 어떤 점이 다를까요? (이 타입이 크레이트의 `stream` 부분에 속해 있다는 점은 신경 쓰지 않아도 됩니다. 어떤 future 컬렉션과도 잘 동작합니다.)

[dyn]: https://doc.rust-lang.org/book/ch12-03-improving-error-handling-and-modularity.html
[enum-alt]: https://doc.rust-lang.org/book/ch08-01-vectors.html#using-an-enum-to-store-multiple-types
[async-program]: https://doc.rust-lang.org/book/ch17-01-futures-and-syntax.html#our-first-async-program
[iterator-trait]: https://doc.rust-lang.org/book/ch13-02-iterators.html#the-iterator-trait-and-the-next-method