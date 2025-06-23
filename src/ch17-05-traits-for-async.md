## Async를 위한 트레잇 자세히 보기

<!-- Old headings. Do not remove or links may break. -->

<a id="digging-into-the-traits-for-async"></a>

이번 장 전반에 걸쳐 우리는 `Future`, `Pin`, `Unpin`, `Stream`, 그리고 `StreamExt` 트레잇을 다양한 방식으로 사용해왔습니다. 하지만 지금까지는 이들이 어떻게 동작하는지, 서로 어떻게 맞물리는지에 대한 세부 사항까지 깊이 들어가지는 않았습니다. 이는 대부분의 일상적인 Rust 작업에서는 충분합니다. 하지만 때로는 이러한 세부 사항을 좀 더 이해해야 하는 상황이 생기기도 합니다. 이 절에서는 그런 경우에 도움이 될 만큼만 살펴보고, *정말* 깊은 내용은 다른 문서에 남겨두겠습니다.

<!-- Old headings. Do not remove or links may break. -->

<a id="future"></a>

### `Future` 트레잇

이제 `Future` 트레잇이 어떻게 동작하는지 좀 더 자세히 살펴보겠습니다. Rust에서는 이를 다음과 같이 정의합니다:

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

그 트레잇 정의에는 여러 새로운 타입과 지금까지 본 적 없는 문법이 포함되어 있으니, 정의를 하나씩 살펴보겠습니다.

먼저, `Future`의 연관 타입인 `Output`은 해당 future가 어떤 값으로 완결되는지를 나타냅니다. 이는 `Iterator` 트레잇의 연관 타입인 `Item`과 유사합니다. 두 번째로, `Future`에는 `poll` 메서드가 있는데, 이 메서드는 `self` 파라미터로 특별한 `Pin` 참조와 `Context` 타입에 대한 가변 참조를 받고, `Poll<Self::Output>`을 반환합니다. `Pin`과 `Context`에 대해서는 곧 더 자세히 다루겠습니다. 지금은 메서드가 반환하는 `Poll` 타입에 집중해 봅시다:

```rust
enum Poll<T> {
    Ready(T),
    Pending,
}
```

이 `Poll` 타입은 `Option`과 비슷합니다. 값이 있는 `Ready(T)` 변형과 값이 없는 `Pending` 변형이 있습니다. 하지만 `Poll`은 `Option`과는 전혀 다른 의미를 가집니다! `Pending` 변형은 future가 아직 해야 할 작업이 남아 있음을 나타내므로, 호출자는 나중에 다시 확인해야 합니다. `Ready` 변형은 future의 작업이 끝났고 `T` 값을 사용할 수 있음을 의미합니다.

> 참고: 대부분의 future에서는, `Ready`를 반환한 이후에는 호출자가 `poll`을 다시 호출해서는 안 됩니다. 많은 future들은 준비 완료(ready) 상태가 된 후에 다시 `poll`을 호출하면 패닉이 발생합니다. 다시 호출해도 안전한 future의 경우, 해당 내용이 문서에 명시되어 있습니다. 이는 `Iterator::next`의 동작 방식과 유사합니다.

`await`를 사용하는 코드를 보면, Rust는 내부적으로 해당 코드를 `poll`을 호출하는 코드로 컴파일합니다. Listing 17-4를 다시 보면, 단일 URL의 페이지 제목을 출력하는 예제가 있었는데, Rust는 이를 대략(정확히는 아니지만) 다음과 같은 코드로 컴파일합니다:

```rust,ignore
match page_title(url).poll() {
    Ready(page_title) => match page_title {
        Some(title) => println!("The title for {url} was {title}"),
        None => println!("{url} had no title"),
    }
    Pending => {
        // But what goes here?
    }
}
```

future가 아직 `Pending` 상태라면 어떻게 해야 할까요? future가 준비될 때까지 계속해서 다시 시도할 방법이 필요합니다. 다시 말해, 반복문이 필요합니다:

```rust,ignore
let mut page_title_fut = page_title(url);
loop {
    match page_title_fut.poll() {
        Ready(value) => match page_title {
            Some(title) => println!("The title for {url} was {title}"),
            None => println!("{url} had no title"),
        }
        Pending => {
            // continue
        }
    }
}
```

만약 Rust가 정말로 위의 코드처럼 컴파일된다면, 모든 `await`가 블로킹이 되어버릴 것입니다. 이는 우리가 원했던 비동기 동작과는 정반대입니다! 대신 Rust는 루프가 이 future의 작업을 잠시 멈추고, 다른 future의 작업을 처리한 뒤 다시 이 future를 확인할 수 있도록 제어권을 넘길 수 있게 만듭니다. 앞서 살펴본 것처럼, 이러한 역할을 수행하는 것이 바로 async 런타임이며, 이러한 스케줄링과 조정 작업이 런타임의 주요 임무 중 하나입니다.

이 장의 앞부분에서 `rx.recv`를 기다리는 과정을 설명했습니다. `recv` 호출은 future를 반환하고, 이 future를 await하면 내부적으로 poll이 호출됩니다. 런타임은 future가 준비될 때까지, 즉 채널이 닫힐 때 `None`이 반환되거나 메시지가 도착해 `Some(message)`가 반환될 때까지 future를 일시 중지시킵니다. 이제 `Future` 트레잇, 특히 `Future::poll`에 대해 더 깊이 이해했으니, 이 동작이 어떻게 이루어지는지 알 수 있습니다. 런타임은 future가 `Poll::Pending`을 반환하면 아직 준비되지 않았음을 인식합니다. 반대로, `poll`이 `Poll::Ready(Some(message))` 또는 `Poll::Ready(None)`을 반환하면 future가 준비되었음을 알고 다음 단계로 진행합니다.

런타임이 실제로 이 작업을 어떻게 수행하는지는 이 책의 범위를 벗어나지만, 중요한 점은 future의 기본 동작 방식입니다. 즉, 런타임은 자신이 관리하는 각 future를 _poll_ 하며, 아직 준비되지 않은 future는 다시 잠들게 둡니다.

<!-- Old headings. Do not remove or links may break. -->

<a id="pinning-and-the-pin-and-unpin-traits"></a>

### `Pin`과 `Unpin` 트레잇

Listing 17-16에서 pinning 개념을 소개할 때, 우리는 꽤 복잡한 에러 메시지를 만났습니다. 다시 한 번 그 관련 부분을 살펴보겠습니다:

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-16
cargo build
copy *only* the final `error` block from the errors
-->

```text
error[E0277]: `{async block@src/main.rs:10:23: 10:33}` cannot be unpinned
  --> src/main.rs:48:33
   |
48 |         trpl::join_all(futures).await;
   |                                 ^^^^^ the trait `Unpin` is not implemented for `{async block@src/main.rs:10:23: 10:33}`
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
   = note: required for `Box<{async block@src/main.rs:10:23: 10:33}>` to implement `Future`
note: required by a bound in `futures_util::future::join_all::JoinAll`
  --> file:///home/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/futures-util-0.3.30/src/future/join_all.rs:29:8
   |
27 | pub struct JoinAll<F>
   |            ------- required by a bound in this struct
28 | where
29 |     F: Future,
   |        ^^^^^^ required by this bound in `JoinAll`
```
이 에러 메시지는 우리가 값을 pin 해야 한다는 것뿐만 아니라, pinning이 왜 필요한지도 알려줍니다. `trpl::join_all` 함수는 `JoinAll`이라는 구조체를 반환합니다. 이 구조체는 제네릭 타입 `F`를 가지며, 이 타입은 `Future` 트레잇을 구현해야 한다는 제약이 있습니다. future를 `await`로 직접 기다릴 때는 Rust가 내부적으로 해당 future를 자동으로 pin 처리해줍니다. 그래서 우리가 future를 await할 때마다 매번 `pin!`을 사용할 필요가 없는 것입니다.

하지만 여기서는 future를 직접 await하는 것이 아닙니다. 대신 여러 future의 컬렉션을 `join_all` 함수에 전달해서 새로운 future인 `JoinAll`을 만듭니다. `join_all`의 시그니처는 컬렉션의 모든 아이템 타입이 `Future` 트레잇을 구현해야 한다고 요구합니다. 그리고 `Box<T>`가 `Future`를 구현하려면, 그 안에 감싸진 `T`가 `Unpin` 트레잇을 구현하는 future여야만 합니다.

정보가 한꺼번에 많이 나왔죠! 이 내용을 제대로 이해하려면, _pinning_ 과 관련해서 `Future` 트레잇이 실제로 어떻게 동작하는지 좀 더 깊이 살펴볼 필요가 있습니다.

다시 한 번 `Future` 트레잇의 정의를 살펴봅시다:

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
    type Output;

    // Required method
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

`cx` 파라미터와 그 타입인 `Context`는 런타임이 어떻게 실제로 각 future를 언제 확인해야 할지(여전히 lazy하게) 알 수 있게 해주는 핵심입니다. 다시 말하지만, 그 동작의 세부 사항은 이 장의 범위를 벗어나며, 일반적으로는 커스텀 `Future`를 직접 구현할 때만 신경 쓰면 됩니다. 대신 여기서는 `self`의 타입에 집중하겠습니다. 이 메서드는 `self`에 타입 어노테이션이 붙은 것을 처음 보는 예시입니다. `self`에 대한 타입 어노테이션은 다른 함수 파라미터의 타입 어노테이션과 비슷하게 동작하지만, 두 가지 중요한 차이점이 있습니다:

- 이 어노테이션은 해당 메서드를 호출할 때 `self`가 어떤 타입이어야 하는지 Rust에게 알려줍니다.

- 아무 타입이나 올 수 있는 것이 아니라, 메서드가 구현된 타입 자체, 그 타입에 대한 참조나 스마트 포인터, 또는 그 타입에 대한 참조를 감싼 `Pin`만 허용됩니다.

이 문법에 대해서는 [18장][ch-18]<!-- ignore -->에서 더 자세히 다룹니다. 지금은 future가 `Pending`인지 `Ready(Output)`인지 확인하기 위해 poll을 호출하려면, 해당 타입에 대한 `Pin`으로 감싼 가변 참조가 필요하다는 것만 알아두면 충분합니다.

`Pin`은 `&`, `&mut`, `Box`, `Rc`와 같은 포인터 유사 타입을 감싸는 래퍼입니다. (엄밀히 말하면, `Pin`은 `Deref` 또는 `DerefMut` 트레잇을 구현한 타입과 함께 동작하지만, 이는 사실상 포인터 타입과만 동작하는 것과 같습니다.) `Pin` 자체는 포인터가 아니며, `Rc`나 `Arc`처럼 참조 카운팅과 같은 동작을 제공하지도 않습니다. 오로지 컴파일러가 포인터 사용에 제약을 강제할 수 있도록 도와주는 도구일 뿐입니다.

`await`가 내부적으로 `poll` 호출로 구현된다는 점을 떠올리면, 앞서 봤던 에러 메시지가 왜 `Unpin`과 관련되어 있었는지 조금은 이해할 수 있습니다. 그렇다면 `Pin`은 `Unpin`과 정확히 어떤 관계가 있고, 왜 `Future`에서 `poll`을 호출하려면 `self`가 반드시 `Pin` 타입이어야 할까요?

이 장 앞부분에서 살펴본 것처럼, 하나의 future 안에 여러 개의 await 지점이 있으면, 컴파일러는 이를 상태 기계(state machine)로 변환합니다. 그리고 이 상태 기계가 Rust의 일반적인 안전성 규칙(빌림, 소유권 등)을 모두 따르도록 만듭니다. 이를 위해 Rust는 각 await 지점 사이(혹은 async 블록의 끝까지) 어떤 데이터가 필요한지 분석합니다. 그리고 소스 코드의 해당 구간에서 사용될 데이터에 맞춰 상태 기계의 각 변형(variant)을 생성합니다. 각 변형은 그 구간에서 필요한 데이터에 접근할 수 있도록, 해당 데이터를 소유하거나, 가변/불변 참조를 얻도록 설계됩니다.

지금까지는 문제가 없었습니다. 어떤 async 블록에서 소유권이나 참조에 대해 잘못 작성하면, borrow checker가 오류를 알려줍니다. 하지만 해당 블록에 대응하는 future를 이리저리 옮겨야 할 때—예를 들어, `join_all`에 전달하기 위해 future를 `Vec`에 넣거나, future를 함수에서 반환하려고 할 때—상황이 더 복잡해집니다.

future를 옮긴다는 것은, Rust가 우리를 위해 생성한 상태 기계(state machine) 자체를 옮긴다는 의미입니다. 그리고 Rust의 대부분 타입과 달리, async 블록에서 생성된 future는 각 변형(variant)의 필드에 자기 자신을 참조하는 내부 참조를 가질 수 있습니다. 이는 Figure 17-4의 단순화된 그림에서 볼 수 있습니다.

<figure>

<img alt="단일 열, 세 개 행으로 이루어진 테이블로, future인 fut1을 나타냅니다. 첫 번째와 두 번째 행에는 데이터 값 0과 1이 들어 있고, 세 번째 행에서는 두 번째 행을 가리키는 화살표가 있어 future 내부의 자기 참조를 표현합니다." src="img/trpl17-04.svg" class="center" />

<figcaption>Figure 17-4: 자기 자신을 참조하는 데이터 타입.</figcaption>

</figure>

하지만 기본적으로 자기 자신에 대한 참조를 가진 객체는 이동하는 것이 안전하지 않습니다. 참조는 항상 자신이 가리키는 실제 메모리 주소를 가리키기 때문입니다(그림 17-5 참고). 만약 데이터 구조체 자체를 이동시키면, 내부 참조들은 이전 위치를 계속 가리키게 됩니다. 그러나 그 메모리 위치는 이제 유효하지 않습니다. 한 가지 문제는, 데이터 구조체를 변경해도 그 값이 업데이트되지 않는다는 점입니다. 더 중요한 문제는, 컴퓨터가 이제 그 메모리 공간을 다른 용도로 자유롭게 재사용할 수 있다는 것입니다! 나중에 완전히 무관한 데이터를 읽게 될 수도 있습니다.

<figure>

<img alt="두 개의 테이블이 나란히 배치되어 있으며, 각각 fut1과 fut2라는 두 future를 나타냅니다. 각 테이블은 한 개의 열과 세 개의 행으로 구성되어 있으며, fut1에서 fut2로 future가 이동된 결과를 보여줍니다. 첫 번째 테이블인 fut1은 회색으로 처리되어 있고 각 인덱스에는 물음표가 있어, 알 수 없는 메모리 상태를 나타냅니다. 두 번째 테이블인 fut2는 첫 번째와 두 번째 행에 각각 0과 1이 들어 있고, 세 번째 행에서는 fut1의 두 번째 행을 가리키는 화살표가 있어 future가 이동되기 전의 이전 메모리 위치를 참조하고 있음을 나타냅니다." src="img/trpl17-05.svg" class="center" />

<figcaption>Figure 17-5: 자기 참조 데이터 타입을 이동시켰을 때 발생하는 안전하지 않은 결과</figcaption>

</figure>

이론적으로는, 러스트 컴파일러가 어떤 객체가 이동될 때마다 그 객체를 참조하는 모든 참조를 업데이트할 수도 있습니다. 하지만 참조가 복잡하게 얽혀 있다면, 이런 작업은 성능에 큰 부담을 줄 수 있습니다. 만약 해당 데이터 구조가 _메모리에서 이동하지 않도록_ 보장할 수 있다면, 참조를 전혀 업데이트할 필요가 없을 것입니다. 바로 이것이 러스트의 빌림 검사기가 요구하는 사항입니다. 안전한 코드에서는, 활성 참조가 존재하는 아이템을 이동시키는 것을 방지합니다.

`Pin`은 이러한 원칙을 바탕으로 우리가 필요한 정확한 보증을 제공합니다. 값을 포인터로 감싼 뒤 `Pin`으로 _핀_ 하면, 그 값은 더 이상 이동할 수 없습니다. 즉, `Pin<Box<SomeType>>`이 있다면, 실제로 핀되는 것은 `Box` 포인터가 아니라 `SomeType` 값입니다. Figure 17-6은 이 과정을 보여줍니다.

<figure>

<img alt="세 개의 박스가 나란히 배치되어 있습니다. 첫 번째는 'Pin', 두 번째는 'b1', 세 번째는 'pinned'로 라벨링되어 있습니다. 'pinned' 안에는 'fut'라는 테이블이 있으며, 단일 열로 구성되어 있습니다. 이 테이블은 데이터 구조의 각 부분을 나타내는 셀들로 이루어진 future를 표현합니다. 첫 번째 셀에는 값 '0'이 들어 있고, 두 번째 셀에서는 화살표가 나와 네 번째이자 마지막 셀(값 '1')을 가리키고 있습니다. 세 번째 셀에는 점선과 생략 부호(...)가 있어 데이터 구조에 다른 부분이 있을 수 있음을 나타냅니다. 전체적으로 'fut' 테이블은 자기 자신을 참조하는 future를 나타냅니다. 'Pin' 박스에서 시작된 화살표는 'b1' 박스를 거쳐 'pinned' 박스 안의 'fut' 테이블에 도달합니다." src="img/trpl17-06.svg" class="center" />

<figcaption>Figure 17-6: 자기 참조 future 타입을 가리키는 `Box`를 pinning하는 모습.</figcaption>

</figure>

사실 `Box` 포인터 자체는 자유롭게 이동할 수 있습니다. 중요한 것은, 궁극적으로 참조되는 데이터가 제자리에 유지되는지 여부입니다. 포인터가 이동하더라도, _그 포인터가 가리키는 데이터가 같은 위치에 있다면_ (Figure 17-7과 같이) 아무런 문제가 없습니다. 별도의 연습 문제로, 각 타입과 `std::pin` 모듈의 문서를 참고하여 `Box`를 감싼 `Pin`을 사용해 이 과정을 어떻게 구현할 수 있을지 고민해 보세요. 핵심은 자기 참조 타입 자체가 이동할 수 없도록, 즉 핀(pin)되어 있어야 한다는 점입니다.

<figure>

<img alt="세 개의 열로 대략 배치된 네 개의 박스가 있습니다. 이전 다이어그램과 동일하지만 두 번째 열에 변화가 있습니다. 이제 두 번째 열에는 'b1'과 'b2'라는 두 개의 박스가 있고, 'b1'은 회색으로 처리되어 있습니다. 'Pin'에서 시작된 화살표가 'b1'이 아니라 'b2'를 거쳐 'pinned'로 향하는데, 이는 포인터가 'b1'에서 'b2'로 이동했지만 'pinned' 안의 데이터는 이동하지 않았음을 나타냅니다." src="img/trpl17-07.svg" class="center" />

<figcaption>Figure 17-7: 자기 참조 future 타입을 가리키는 `Box`를 이동시킨 모습.</figcaption>

</figure>

하지만 대부분의 타입은 내부적으로 참조를 가지고 있지 않기 때문에, `Pin` 래퍼로 감싸져 있더라도 자유롭게 이동해도 전혀 문제가 없습니다. pinning에 대해 신경 써야 하는 경우는 내부 참조를 가진 아이템에 한정됩니다. 숫자나 불리언과 같은 원시 값들은 내부 참조가 없으므로 안전합니다. 일반적으로 Rust에서 사용하는 대부분의 타입도 마찬가지입니다. 예를 들어, `Vec` 타입은 별다른 걱정 없이 자유롭게 이동할 수 있습니다. 지금까지 살펴본 내용만 놓고 보면, 만약 `Pin<Vec<String>>`이 있다면, 내부적으로 참조가 없더라도 `Pin`이 제공하는 안전하지만 제한적인 API만 사용해야 할 것처럼 보입니다. 하지만 실제로는, 내부 참조가 없는 타입은 자유롭게 이동해도 괜찮다는 사실을 컴파일러에게 알려줄 방법이 필요합니다. 바로 여기서 `Unpin`이 등장합니다.

`Unpin`은 16장에서 살펴본 `Send`와 `Sync`처럼 자체 기능이 없는 마커 트레잇입니다. 마커 트레잇은 특정 트레잇을 구현한 타입이 어떤 상황에서 안전하게 사용될 수 있음을 컴파일러에게 알려주기 위해 존재합니다. `Unpin`은 해당 타입이 이동과 관련된 어떤 제약도 필요하지 않다는 사실을 컴파일러에게 알려줍니다.

<!--
  The inline `<code>` in the next block is to allow the inline `<em>` inside it,
  matching what NoStarch does style-wise, and emphasizing within the text here
  that it is something distinct from a normal type.
-->

`Send`와 `Sync`와 마찬가지로, 컴파일러는 안전하다고 판단할 수 있는 모든 타입에 대해 `Unpin`을 자동으로 구현합니다. 특별한 경우로, 역시 `Send`와 `Sync`와 비슷하게, 어떤 타입에 대해 `Unpin`이 *구현되지 않은* 경우가 있습니다. 이때의 표기법은 <code>impl !Unpin for <em>SomeType</em></code>과 같이 쓰며, 여기서 <code><em>SomeType</em></code>은 해당 타입에 대한 포인터가 `Pin`에서 사용될 때 반드시 그 보증을 지켜야만 안전한 타입임을 의미합니다.

즉, `Pin`과 `Unpin`의 관계에서 기억해야 할 점은 두 가지입니다. 첫째, `Unpin`이 "일반적인" 경우이고, `!Unpin`이 특별한 경우라는 점입니다. 둘째, 어떤 타입이 `Unpin`을 구현하는지(`!Unpin`인지)는 오직 <code>Pin<&mut <em>SomeType</em>></code>과 같이 해당 타입에 대한 핀된 포인터를 사용할 때만 중요하다는 점입니다.

좀 더 구체적으로 살펴보면, 예를 들어 `String`을 생각해봅시다. `String`은 길이와 그 내용을 이루는 유니코드 문자 데이터를 가지고 있습니다. Figure 17-8에서처럼, 우리는 `String`을 `Pin`으로 감쌀 수 있습니다. 하지만 `String`은 Rust의 대부분 타입과 마찬가지로 자동으로 `Unpin`을 구현합니다.

<figure>

<img alt="동시 작업 흐름" src="img/trpl17-08.svg" class="center" />

<figcaption>Figure 17-8: `String`을 pinning한 모습. 점선은 `String`이 `Unpin` 트레잇을 구현하므로 실제로는 pinning되지 않음을 나타냅니다.</figcaption>

</figure>

따라서 `String`이 `!Unpin`을 구현했다면 불법이었을 작업, 즉 Figure 17-9처럼 메모리의 정확히 같은 위치에서 한 문자열을 다른 문자열로 교체하는 것도 문제없이 할 수 있습니다. 이는 `String`이 내부적으로 참조를 가지고 있지 않아 이동해도 안전하기 때문에, `Pin`의 계약을 위반하지 않습니다! 바로 그렇기 때문에 `String`은 `!Unpin`이 아니라 `Unpin`을 구현하는 것입니다.

<figure>

<img alt="동시 작업 흐름" src="img/trpl17-09.svg" class="center" />

<figcaption>Figure 17-9: 메모리에서 `String`을 완전히 다른 `String`으로 교체하는 모습.</figcaption>

</figure>

이제 Listing 17-17에서 `join_all` 호출 시 보고된 에러를 이해할 수 있을 만큼 충분히 배웠습니다. 처음에는 async 블록에서 생성된 future들을 `Vec<Box<dyn Future<Output = ()>>>`로 옮기려고 했지만, 앞서 살펴본 것처럼 이러한 future들은 내부적으로 자기 참조를 가질 수 있으므로 `Unpin`을 구현하지 않습니다. 이 future들은 pinning이 필요하며, pinning을 거친 후에는 `Pin` 타입을 `Vec`에 담아도 future 내부의 데이터가 _이동되지 않음_ 을 확신할 수 있습니다.

`Pin`과 `Unpin`은 주로 저수준 라이브러리를 만들거나, 런타임 자체를 구현할 때 중요합니다. 일상적인 Rust 코드에서는 자주 신경 쓸 일이 없지만, 에러 메시지에서 이 트레잇들을 보게 된다면 이제 어떻게 코드를 고쳐야 할지 더 잘 알 수 있을 것입니다!

> 참고: `Pin`과 `Unpin`의 조합 덕분에, Rust에서는 자기 참조를 가지는 복잡한 타입들도 안전하게 구현할 수 있습니다. `Pin`이 필요한 타입은 오늘날 async Rust에서 가장 흔히 등장하지만, 가끔 다른 맥락에서도 볼 수 있습니다.
>
> `Pin`과 `Unpin`이 실제로 어떻게 동작하는지, 그리고 이들이 지켜야 하는 규칙에 대해서는 `std::pin`의 API 문서에 매우 자세히 설명되어 있으니, 더 깊이 배우고 싶다면 꼭 참고해 보세요.
>
> 내부적으로 어떻게 동작하는지 더 자세히 알고 싶다면, [_Asynchronous Programming in Rust_][async-book]의 [2장][under-the-hood]과 [4장][pinning]을 참고하세요.

### `Stream` 트레잇

이제 `Future`, `Pin`, `Unpin` 트레잇에 대해 더 깊이 이해했으니, 이제는 `Stream` 트레잇에 대해 살펴볼 차례입니다. 이 장의 앞부분에서 배운 것처럼, 스트림은 비동기 반복자와 유사합니다. 하지만 `Iterator`나 `Future`와 달리, 이 글을 쓰는 시점 기준으로 표준 라이브러리에는 `Stream`의 정의가 없습니다. 대신, 생태계 전반에서 널리 사용되는 `futures` 크레이트의 매우 일반적인 정의가 존재합니다.

먼저, `Stream` 트레잇이 어떻게 두 트레잇의 개념을 결합하는지 보기 전에, `Iterator`와 `Future` 트레잇의 정의를 다시 살펴봅시다. `Iterator`에서는 시퀀스라는 개념이 있습니다. `next` 메서드는 `Option<Self::Item>`을 제공합니다. `Future`에서는 시간에 따라 준비되는 값이라는 개념이 있습니다. `poll` 메서드는 `Poll<Self::Output>`을 반환합니다. 시간이 지나면서 준비되는 여러 아이템의 시퀀스를 표현하기 위해, 이 두 가지 특징을 결합한 `Stream` 트레잇을 정의할 수 있습니다:

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

trait Stream {
    type Item;

    fn poll_next(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>
    ) -> Poll<Option<Self::Item>>;
}
```

`Stream` 트레잇은 스트림이 생성하는 아이템의 타입을 나타내는 연관 타입 `Item`을 정의합니다. 이는 `Iterator`와 유사하게, 0개 이상의 아이템이 있을 수 있다는 점에서 비슷하며, 항상 하나의 `Output`만 가지는 `Future`와는 다릅니다(그 값이 단위 타입 `()`일지라도).

또한 `Stream`은 이러한 아이템을 얻기 위한 메서드를 정의합니다. 이 메서드는 `poll_next`라고 부르는데, 이는 `Future::poll`처럼 폴링(polling) 방식으로 동작하며, `Iterator::next`처럼 아이템의 시퀀스를 생성한다는 점을 명확히 하기 위함입니다. 반환 타입은 `Poll`과 `Option`을 결합한 형태입니다. 바깥쪽 타입은 `Poll`로, future처럼 준비 여부를 확인해야 하기 때문입니다. 안쪽 타입은 `Option`으로, 반복자처럼 더 많은 메시지가 있는지 신호를 주기 위함입니다.

이와 매우 유사한 정의가 앞으로 Rust 표준 라이브러리의 일부가 될 가능성이 높습니다. 그 전까지는 대부분의 런타임에서 이 트레잇을 제공하므로, 이를 신뢰하고 사용할 수 있습니다. 이후에 다루는 내용도 일반적으로 모두 적용됩니다!

스트리밍 섹션에서 살펴본 예제에서는 사실 `poll_next`나 `Stream`을 직접 사용하지 않고, 대신 `next`와 `StreamExt`를 사용했습니다. 물론 직접 `poll_next` API를 사용해 직접 `Stream` 상태 기계를 작성할 수도 있고, 마찬가지로 future의 `poll` 메서드를 직접 사용할 수도 있습니다. 하지만 `await`를 사용하는 것이 훨씬 편리하며, `StreamExt` 트레잇이 `next` 메서드를 제공해주기 때문에 이를 활용할 수 있습니다.

```rust
{{#rustdoc_include ../listings/ch17-async-await/no-listing-stream-ext/src/lib.rs:here}}
```

<!--
TODO: update this if/when tokio/etc. update their MSRV and switch to using async functions
in traits, since the lack thereof is the reason they do not yet have this.
-->

> 참고: 이 장 앞부분에서 사용한 실제 정의는 약간 다릅니다. 이는 트레잇에서 async 함수를 아직 지원하지 않는 Rust 버전도 지원하기 위해서입니다. 그래서 실제로는 다음과 같이 생겼습니다:
>
> ```rust,ignore
> fn next(&mut self) -> Next<'_, Self> where Self: Unpin;
> ```
>
> 여기서 `Next` 타입은 `Future`를 구현하는 구조체이며, `Next<'_, Self>`와 같이 `self`에 대한 참조의 라이프타임을 명시할 수 있게 해줍니다. 덕분에 이 메서드에서 `await`를 사용할 수 있습니다.

`StreamExt` 트레잇에는 스트림에서 사용할 수 있는 다양한 유용한 메서드들이 정의되어 있습니다. `StreamExt`는 `Stream`을 구현하는 모든 타입에 자동으로 구현되지만, 이 두 트레잇을 분리해서 정의한 이유는 핵심 트레잇에 영향을 주지 않고 커뮤니티가 편의 API를 자유롭게 발전시킬 수 있도록 하기 위함입니다.

`trpl` 크레이트에서 사용하는 `StreamExt` 버전은 `next` 메서드뿐만 아니라, `Stream::poll_next`를 올바르게 호출하는 `next`의 기본 구현도 제공합니다. 즉, 직접 스트리밍 데이터 타입을 만들어야 할 때에도 _오직_ `Stream`만 구현하면 되고, 그 타입을 사용하는 사람은 자동으로 `StreamExt`의 다양한 메서드를 활용할 수 있습니다.

이제 이 트레잇들의 저수준 동작에 대한 설명은 여기까지입니다. 마지막으로, future(스트림 포함), 태스크, 그리고 스레드가 어떻게 함께 동작하는지 살펴보겠습니다!

[ch-18]: https://doc.rust-lang.org/book/ch18-00-oop.html
[async-book]: https://rust-lang.github.io/async-book/
[under-the-hood]: https://rust-lang.github.io/async-book/02_execution/01_chapter.html
[pinning]: https://rust-lang.github.io/async-book/04_pinning/01_chapter.html
[first-async]: https://doc.rust-lang.org/book/ch17-01-futures-and-syntax.html#our-first-async-program
[any-number-futures]: https://doc.rust-lang.org/book/ch17-03-more-futures.html#working-with-any-number-of-futures