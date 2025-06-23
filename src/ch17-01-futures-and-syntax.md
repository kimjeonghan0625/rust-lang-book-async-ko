## Future와 Async 문법
Rust에서 비동기 프로그래밍의 핵심 요소는 *future*와 Rust의 `async`, `await` 키워드입니다.

*future*란 지금은 준비되지 않았지만 언젠가는 준비될 값입니다. (이와 비슷한 개념은 여러 언어에서 *task*나 *promise* 등 다양한 이름으로 등장합니다.) Rust는 다양한 비동기 연산을 서로 다른 데이터 구조로 구현하더라도 공통된 인터페이스를 제공할 수 있도록 `Future` 트레이트를 제공합니다. Rust에서 future는 `Future` 트레이트를 구현하는 타입입니다. 각 future는 자신만의 진행 상황과 "준비됨"이 무엇을 의미하는지에 대한 정보를 가지고 있습니다.

`async` 키워드는 블록이나 함수에 적용하여 해당 코드가 중단되고 다시 재개될 수 있음을 명시합니다. async 블록이나 async 함수 내부에서는 `await` 키워드를 사용해 *future를 기다릴* 수 있습니다(즉, future가 준비될 때까지 대기합니다). async 블록이나 함수 내에서 future를 await하는 모든 지점은 해당 블록이나 함수가 일시 중지되고 다시 시작될 수 있는 잠재적인 위치입니다. future가 값을 사용할 수 있는지 확인하는 과정을 *폴링(polling)* 이라고 합니다.

C#나 JavaScript와 같은 다른 언어들도 비동기 프로그래밍을 위해 `async`와 `await` 키워드를 사용합니다. 만약 이러한 언어에 익숙하다면, Rust가 문법을 처리하는 방식 등에서 상당한 차이점이 있다는 것을 알 수 있을 것입니다. 이는 충분한 이유가 있기 때문이며, 곧 그 이유를 살펴보겠습니다!

비동기 Rust 코드를 작성할 때는 대부분 `async`와 `await` 키워드를 사용합니다. Rust는 이 키워드들을 `Future` 트레이트를 활용하는 동등한 코드로 컴파일합니다. 이는 Rust가 `for` 루프를 `Iterator` 트레이트를 사용하는 코드로 변환하는 것과 비슷합니다. Rust는 `Future` 트레이트를 제공하기 때문에, 필요하다면 여러분만의 데이터 타입에 대해 직접 구현할 수도 있습니다. 이 장에서 살펴볼 많은 함수들은 각자 자신만의 `Future` 구현을 반환합니다. 트레이트의 정의와 동작 방식에 대해서는 이 장의 마지막에서 다시 다루겠지만, 지금은 이 정도만 알아도 앞으로 내용을 이해하는 데 충분합니다.

이 모든 내용이 다소 추상적으로 느껴질 수 있으니, 실제로 첫 번째 async 프로그램을 작성해 보겠습니다. 간단한 웹 스크래퍼를 만들어볼 텐데요, 명령줄에서 두 개의 URL을 입력받아 둘 다 동시에 가져오고, 먼저 완료되는 결과를 반환할 것입니다. 이 예제에는 새로운 문법이 꽤 등장하지만 걱정하지 마세요—진행하면서 필요한 모든 내용을 하나씩 설명해드리겠습니다.

## 우리의 첫 번째 Async 프로그램

이 장에서는 Rust 비동기 생태계의 다양한 부분을 다루기보다는 async 자체를 배우는 데 집중할 수 있도록, `trpl` 크레이트를 준비했습니다. (`trpl`은 “The Rust Programming Language”의 약자입니다.) 이 크레이트는 주로 [`futures`][futures-crate]<!-- ignore -->와 [`tokio`][tokio]<!-- ignore --> 크레이트에서 필요한 타입, 트레이트, 함수들을 모두 재내보냅니다. `futures` 크레이트는 Rust의 공식적인 비동기 실험 공간이며, 실제로 `Future` 트레이트가 처음 설계된 곳이기도 합니다. Tokio는 현재 Rust에서 가장 널리 사용되는 비동기 런타임으로, 특히 웹 애플리케이션에서 많이 쓰입니다. 이 외에도 훌륭한 런타임들이 많으며, 여러분의 목적에 더 적합한 것이 있을 수도 있습니다. 우리는 `trpl`의 내부 구현에 `tokio` 크레이트를 사용하는데, 이는 충분히 검증되었고 널리 사용되기 때문입니다.

일부 경우에는, `trpl`이 원래의 API 이름을 바꾸거나 감싸서 이 장에서 중요한 부분에 집중할 수 있도록 했습니다. 크레이트가 실제로 어떤 일을 하는지 궁금하다면 [소스 코드][crate-source]<!-- ignore -->를 직접 확인해 보시길 권장합니다. 각 재내보내기가 어떤 크레이트에서 왔는지 확인할 수 있고, 크레이트의 동작에 대해 자세한 주석도 남겨두었습니다.

이제 `hello-async`라는 새 바이너리 프로젝트를 만들고, `trpl` 크레이트를 의존성에 추가해봅시다:

```console
$ cargo new hello-async
$ cd hello-async
$ cargo add trpl
```

이제 `trpl`이 제공하는 다양한 요소들을 활용해 우리의 첫 번째 async 프로그램을 작성해볼 수 있습니다. 두 개의 웹 페이지를 가져와 각각의 `<title>` 요소를 추출하고, 이 모든 과정을 가장 먼저 끝낸 페이지의 제목을 출력하는 간단한 커맨드라인 도구를 만들어보겠습니다.

### page_title 함수 정의하기

먼저, 하나의 페이지 URL을 매개변수로 받아 요청을 보내고, 해당 페이지의 `<title>` 요소의 텍스트를 반환하는 함수를 작성해봅시다(Listing 17-1 참고).

<Listing number="17-1" file-name="src/main.rs" caption="HTML 페이지에서 title 요소를 가져오는 async 함수 정의하기">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-01/src/main.rs:all}}
```

</Listing>

먼저, `page_title`라는 이름의 함수를 정의하고 `async` 키워드로 표시합니다. 그런 다음, 전달받은 URL을 가져오기 위해 `trpl::get` 함수를 사용하고, 응답을 기다리기 위해 `await` 키워드를 추가합니다. 응답의 텍스트를 얻기 위해서는 해당 응답의 `text` 메서드를 호출하고, 다시 한 번 `await` 키워드로 기다립니다. 이 두 단계 모두 비동기적으로 동작합니다. `get` 함수의 경우, 서버가 HTTP 헤더, 쿠키 등과 같은 응답의 첫 부분을 보내줄 때까지 기다려야 하며, 이 정보들은 응답 본문과 별도로 전달될 수 있습니다. 특히 본문이 매우 클 경우, 전체가 도착하는 데 시간이 걸릴 수 있습니다. 우리는 응답의 *전체*가 도착할 때까지 기다려야 하므로, `text` 메서드 역시 async입니다.

이 두 future 모두를 명시적으로 await해야 합니다. Rust에서 future는 *게으르기(lazy)* 때문에, `await` 키워드를 사용해 요청하기 전까지 아무 일도 하지 않습니다. (실제로 future를 사용하지 않으면 Rust가 컴파일러 경고를 표시합니다.) 이는 13장에서 반복자(iterator)에 대해 다룬 [반복자로 일련의 아이템들 처리하기][iterators-lazy]<!-- ignore --> 부분을 떠올리게 할 수 있습니다. 반복자는 `next` 메서드를 직접 호출하거나, `for` 루프 또는 내부적으로 `next`를 사용하는 `map` 같은 메서드를 통해 호출하지 않는 한 아무 일도 하지 않습니다. 마찬가지로, future도 명시적으로 요청하지 않으면 아무 일도 하지 않습니다. 이러한 게으름(laziness) 덕분에 Rust는 실제로 필요할 때까지 비동기 코드를 실행하지 않을 수 있습니다.

> 참고: 이는 이전 장에서 [스레드 생성하기][thread-spawn]<!--ignore-->에서 `thread::spawn`을 사용할 때와는 다른 동작입니다. 그때는 다른 스레드에 전달한 클로저가 즉시 실행되기 시작했습니다. 또한, 많은 다른 언어에서 async를 다루는 방식과도 다릅니다. 하지만 Rust가 반복자에서와 마찬가지로 성능 보장을 제공하기 위해서는 이러한 동작이 중요합니다.

`response_text`를 얻은 후에는 `Html::parse`를 사용해 이를 `Html` 타입의 인스턴스로 파싱할 수 있습니다. 이제 단순한 문자열이 아니라, HTML을 더 풍부한 데이터 구조로 다룰 수 있는 타입을 갖게 됩니다. 특히, `select_first` 메서드를 사용하면 주어진 CSS 셀렉터에 해당하는 첫 번째 요소를 찾을 수 있습니다. 문자열 `"title"`을 전달하면, 문서 내에 `<title>` 요소가 있다면 그 중 첫 번째 요소를 가져옵니다. 일치하는 요소가 없을 수도 있기 때문에, `select_first`는 `Option<ElementRef>`를 반환합니다. 마지막으로, `Option`에 값이 있을 때만 해당 값으로 작업할 수 있도록 해주는 `Option::map` 메서드를 사용합니다. (여기서 `match` 표현식을 사용할 수도 있지만, `map`이 더 관용적입니다.) `map`에 전달하는 함수의 본문에서는 `title_element`에 대해 `inner_html`을 호출해 그 내용을 `String`으로 얻습니다. 이 모든 과정을 거치면, 최종적으로 `Option<String>`을 얻게 됩니다.

Rust의 `await` 키워드는 기다릴 표현식 *뒤에* 위치한다는 점에 주목하세요. 즉, *접미사(postfix)* 키워드입니다. 만약 다른 언어에서 `async`를 사용해본 경험이 있다면 이 부분이 다르게 느껴질 수 있지만, Rust에서는 메서드 체이닝을 훨씬 더 깔끔하게 작성할 수 있게 해줍니다. 그 결과, Listing 17-2에서 보듯이 `page_title` 함수의 본문에서 `trpl::get`과 `text` 함수 호출을 `await`로 연결해 체이닝할 수 있습니다.

<Listing number="17-2" file-name="src/main.rs" caption="`await` 키워드로 체이닝하기">

```rust
{{#rustdoc_include ../listings/ch17-async-await/listing-17-02/src/main.rs:chaining}}
```

</Listing>

이렇게 해서 우리의 첫 번째 async 함수를 성공적으로 작성했습니다! 이제 `main`에서 이 함수를 호출하는 코드를 추가하기 전에, 우리가 작성한 코드가 의미하는 바를 조금 더 살펴보겠습니다.

Rust가 `async` 키워드가 붙은 블록을 만나면, 해당 블록을 `Future` 트레이트를 구현하는 고유하고 익명인 데이터 타입으로 컴파일합니다. 함수에 `async`가 붙으면, Rust는 해당 함수를 비동기 블록을 본문으로 갖는 일반 함수로 컴파일합니다. async 함수의 반환 타입은 컴파일러가 해당 async 블록을 위해 생성한 익명 타입이 됩니다.

즉, `async fn`을 작성하는 것은 반환 타입의 *future*를 반환하는 함수를 작성하는 것과 같습니다. 컴파일러 입장에서는 Listing 17-1의 `async fn page_title`과 같은 함수 정의는 아래와 같이 비동기 함수가 아닌 형태로 정의한 것과 동등합니다:

```rust
# extern crate trpl; // required for mdbook test
use std::future::Future;
use trpl::Html;

fn page_title(url: &str) -> impl Future<Output = Option<String>> {
    async move {
        let text = trpl::get(url).await.text().await;
        Html::parse(&text)
            .select_first("title")
            .map(|title| title.inner_html())
    }
}
```
변환된 버전의 각 부분을 살펴보겠습니다:

- 10장에서 [“매개변수로서의 트레이트”][impl-trait]<!-- ignore -->에서 다뤘던 `impl Trait` 문법을 사용합니다.
- 반환되는 트레이트는 `Output`이라는 연관 타입을 가진 `Future`입니다. `Output` 타입이 `Option<String>`인 점에 주목하세요. 이는 `async fn` 버전의 `page_title`이 반환하던 타입과 동일합니다.
- 원래 함수 본문에서 호출하던 모든 코드는 `async move` 블록으로 감싸집니다. 블록도 표현식이므로, 이 전체 블록이 함수에서 반환하는 표현식이 됩니다.
- 이 async 블록은 앞서 설명한 대로 `Option<String>` 타입의 값을 생성합니다. 이 값이 반환 타입의 `Output` 타입과 일치합니다. 이는 여러분이 이미 본 다른 블록들과 동일한 방식입니다.
- 새로운 함수 본문이 `async move` 블록인 이유는 `url` 파라미터의 사용 방식 때문입니다. (`async`와 `async move`의 차이점에 대해서는 이 장의 뒷부분에서 더 자세히 다룹니다.)

이제 `main`에서 `page_title`을 호출할 수 있습니다.

## 단일 페이지의 제목 가져오기

먼저, 한 페이지만의 제목을 가져와 보겠습니다. Listing 17-3에서는 12장에서 [커맨드라인 인자 받기][cli-args]<!-- ignore --> 절에서 사용했던 것과 동일한 패턴으로 커맨드라인 인자를 받아옵니다. 그런 다음 첫 번째 URL을 `page_title` 함수에 전달하고, 결과를 await합니다. future가 반환하는 값은 `Option<String>`이므로, 해당 페이지에 `<title>`이 있는지 여부에 따라 다른 메시지를 출력하기 위해 `match` 표현식을 사용합니다.

<Listing number="17-3" file-name="src/main.rs" caption="사용자가 입력한 인자를 받아 `main`에서 `page_title` 함수를 호출하기">

```rust,ignore,does_not_compile
{{#rustdoc_include ../listings/ch17-async-await/listing-17-03/src/main.rs:main}}
```

</Listing>

안타깝게도 이 코드는 컴파일되지 않습니다. `await` 키워드는 오직 async 함수나 블록 안에서만 사용할 수 있으며, Rust에서는 특별한 함수인 `main`에 `async`를 붙이는 것을 허용하지 않습니다.

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-03
cargo build
copy just the compiler error
-->

```text
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:6:1
  |
6 | async fn main() {
  | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

`main` 함수에 `async`를 붙일 수 없는 이유는, 비동기 코드는 *런타임(runtime)* 이 필요하기 때문입니다. 런타임이란 비동기 코드를 실제로 실행하는 세부 사항을 관리하는 Rust 크레이트입니다. 프로그램의 `main` 함수는 런타임을 *초기화*할 수는 있지만, 그 자체가 런타임은 아닙니다. (이유에 대해서는 곧 더 자세히 살펴보겠습니다.) 비동기 코드를 실행하는 모든 Rust 프로그램은 최소 한 곳에서 런타임을 설정하고 future를 실행합니다.

비동기를 지원하는 대부분의 언어는 런타임을 내장하고 있지만, Rust는 그렇지 않습니다. 대신, 다양한 용도에 맞춰 서로 다른 트레이드오프를 가진 여러 비동기 런타임이 존재합니다. 예를 들어, 많은 CPU 코어와 대용량 RAM을 가진 고성능 웹 서버와, 단일 코어에 적은 RAM, 힙 할당이 불가능한 마이크로컨트롤러는 요구 사항이 매우 다릅니다. 이러한 런타임을 제공하는 크레이트들은 파일이나 네트워크 I/O와 같은 일반적인 기능의 비동기 버전도 함께 제공합니다.

이 장과 이후의 예제에서는 `trpl` 크레이트의 `run` 함수를 사용할 것입니다. 이 함수는 future를 인자로 받아 끝까지 실행합니다. 내부적으로 `run`을 호출하면 future를 실행할 런타임이 설정되고, future가 완료되면 그 결과 값을 반환합니다.

`page_title`이 반환하는 future를 바로 `run`에 넘길 수도 있습니다. 이 경우 future가 완료되면, Listing 17-3에서 시도했던 것처럼 결과로 받은 `Option<String>`에 대해 match를 사용할 수 있습니다. 하지만 이 장의 대부분 예제(그리고 실제 비동기 코드)에서는 단일 async 함수 호출만 하는 것이 아니라 여러 작업을 수행하므로, 대신 `async` 블록을 넘기고 그 안에서 `page_title`의 결과를 명시적으로 await하는 방식을 사용할 것입니다(Listing 17-4 참고).

<Listing number="17-4" caption="`trpl::run`으로 async 블록을 await하기" file-name="src/main.rs">

<!-- should_panic,noplayground because mdbook test does not pass args -->

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch17-async-await/listing-17-04/src/main.rs:run}}
```

</Listing>

이 코드를 실행하면, 처음에 기대했던 대로 동작하는 것을 확인할 수 있습니다:

<!-- manual-regeneration
cd listings/ch17-async-await/listing-17-04
cargo build # skip all the build noise
cargo run https://www.rust-lang.org
# copy the output here
-->

```console
$ cargo run -- https://www.rust-lang.org
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.05s
     Running `target/debug/async_await 'https://www.rust-lang.org'`
The title for https://www.rust-lang.org was
            Rust Programming Language
```

휴—드디어 제대로 동작하는 async 코드를 작성했습니다! 하지만 두 사이트를 서로 경쟁시키는 코드를 추가하기 전에, 잠시 future가 어떻게 동작하는지 다시 살펴보겠습니다.

각 *await 지점*—즉, 코드에서 `await` 키워드를 사용하는 모든 위치—은(는) 제어권이 런타임에 반환되는 지점을 의미합니다. 이를 가능하게 하려면, Rust는 async 블록에 관련된 상태를 추적해야 하며, 그래야 런타임이 다른 작업을 시작했다가 준비가 되면 다시 처음 작업을 진행할 수 있습니다. 이는 눈에 보이지 않는 상태 기계(state machine)로, 마치 각 await 지점에서 현재 상태를 저장하기 위해 아래와 같은 enum을 직접 작성한 것과 같습니다:

```rust
{{#rustdoc_include ../listings/ch17-async-await/no-listing-state-machine/src/lib.rs:enum}}
```

각 상태 간 전환 코드를 직접 작성하는 것은 번거롭고 오류가 발생하기 쉽습니다. 특히 나중에 더 많은 기능과 상태를 추가해야 할 때 더욱 그렇습니다. 다행히도 Rust 컴파일러는 async 코드에 대한 상태 기계 데이터 구조를 자동으로 생성하고 관리해줍니다. 데이터 구조에 대한 일반적인 소유권 및 차용 규칙도 모두 그대로 적용되며, 컴파일러가 이를 검사하고 유용한 오류 메시지도 제공합니다. 이 장의 뒷부분에서 이러한 오류 메시지의 예시도 살펴볼 예정입니다.

궁극적으로 이 상태 기계를 실제로 실행하는 무언가가 필요하며, 바로 *런타임(runtime)* 이 그 역할을 합니다. (런타임과 관련된 자료를 찾아보면 *익스큐터(executor)* 라는 용어를 접할 수 있는데, 익스큐터는 런타임 중에서 async 코드를 실제로 실행하는 부분을 의미합니다.)

이제 왜 컴파일러가 Listing 17-3에서 `main` 함수 자체를 async 함수로 만드는 것을 막았는지 이해할 수 있습니다. 만약 `main`이 async 함수였다면, `main`이 반환하는 future의 상태 기계를 관리할 무언가가 필요하지만, `main`은 프로그램의 시작점이기 때문입니다! 대신 우리는 `main` 함수에서 `trpl::run` 함수를 호출해 런타임을 설정하고, async 블록이 반환하는 future를 끝까지 실행하도록 했습니다.

> 참고: 일부 런타임에서는 매크로를 제공하여 async `main` 함수를 작성할 *수도 있습니다*. 이러한 매크로는 `async fn main() { ... }`를 일반 `fn main`으로 변환하며, Listing 17-4에서 우리가 직접 했던 것과 동일하게 future를 끝까지 실행하는 함수를 호출하도록 만듭니다. 즉, `trpl::run`이 하는 방식과 같습니다.

이제 지금까지 살펴본 내용을 바탕으로, 실제로 어떻게 동시성 코드를 작성할 수 있는지 함께 살펴보겠습니다.
### 두 URL을 서로 경쟁시키기

Listing 17-5에서는 커맨드라인에서 전달받은 두 개의 URL을 `page_title` 함수에 넘기고, 이 둘을 경쟁(race)시킵니다.

<Listing number="17-5" caption="" file-name="src/main.rs">

<!-- should_panic,noplayground because mdbook does not pass args -->

```rust,should_panic,noplayground
{{#rustdoc_include ../listings/ch17-async-await/listing-17-05/src/main.rs:all}}
```

</Listing>

먼저, 사용자가 입력한 각 URL에 대해 `page_title` 함수를 호출합니다. 이렇게 생성된 future를 각각 `title_fut_1`과 `title_fut_2`에 저장합니다. 이 future들은 아직 아무 일도 하지 않는다는 점을 기억하세요. future는 게으르기 때문에, 우리가 아직 await하지 않았기 때문입니다. 그런 다음 이 future들을 `trpl::race`에 전달하면, 둘 중 어떤 future가 먼저 완료되는지 알려주는 값을 반환합니다.

> 참고: 내부적으로 `race`는 더 일반적인 함수인 `select`를 기반으로 구현되어 있습니다. 실제 Rust 코드에서는 `select` 함수를 더 자주 접하게 될 것입니다. `select` 함수는 `trpl::race` 함수로는 할 수 없는 다양한 작업을 수행할 수 있지만, 그만큼 추가적인 복잡성도 있으므로 지금은 자세히 다루지 않겠습니다.

어느 future가 먼저 완료될지 예측할 수 없으므로, `Result` 타입을 반환하는 것은 적절하지 않습니다. 대신, `race` 함수는 우리가 이전에 본 적 없는 타입인 `trpl::Either`를 반환합니다. `Either` 타입은 두 가지 경우를 가질 수 있다는 점에서 `Result`와 비슷하지만, `Result`와 달리 성공이나 실패의 의미가 내포되어 있지 않습니다. 대신, `Left`와 `Right`를 사용해 “둘 중 하나”임을 나타냅니다:

```rust
enum Either<A, B> {
    Left(A),
    Right(B),
}
```

`race` 함수는 먼저 완료된 future의 출력값을 `Left`로, 두 번째 future가 먼저 완료되면 그 출력값을 `Right`로 반환합니다. 이는 함수 호출 시 인자 순서와 일치합니다. 즉, 첫 번째 인자는 두 번째 인자의 왼쪽에 있으므로, 먼저 끝난 쪽이 `Left`가 됩니다.

또한, `page_title` 함수가 전달받은 URL을 함께 반환하도록 수정했습니다. 이렇게 하면, 먼저 응답이 온 페이지에 `<title>` 요소가 없더라도 어떤 URL이 먼저 완료되었는지 의미 있는 메시지를 출력할 수 있습니다. 이 정보를 바탕으로, `println!` 출력도 어느 URL이 먼저 끝났고 해당 웹 페이지의 `<title>`이 무엇인지(또는 없는 경우) 함께 표시하도록 업데이트했습니다.

이제 여러분은 간단한 웹 스크래퍼를 완성했습니다! 여러 URL을 입력해 명령줄 도구를 실행해 보세요. 어떤 사이트가 항상 더 빠른지, 혹은 실행할 때마다 더 빠른 사이트가 달라지는지 확인할 수 있습니다. 무엇보다도, 이제 future를 다루는 기본기를 익혔으니, 앞으로 async로 할 수 있는 더 깊은 내용들을 배워볼 준비가 되었습니다.

<!-- [impl-trait]: ch10-02-traits.html#traits-as-parameters -->
[impl-trait]: https://doc.rust-lang.org/book/ch10-02-traits.html#traits-as-parameters
[iterators-lazy]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[thread-spawn]: https://doc.rust-lang.org/book/ch16-01-threads.html#creating-a-new-thread-with-spawn
[cli-args]: https://doc.rust-lang.org/book/ch12-01-accepting-command-line-arguments.html


<!-- TODO: map source link version to version of Rust? -->

[crate-source]: https://github.com/rust-lang/book/tree/main/packages/trpl
[futures-crate]: https://crates.io/crates/futures
[tokio]: https://tokio.rs
