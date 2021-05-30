# Окно

> [Ранее](00-setup.html) Мы получили некоторую мотивацию для использования Rust и настроили нашу среду разработки.

В этой главе мы создадим окно! Если это не звучит слишком увлекательно, мы также узнаем о пакетах Rust. Очень волнующе.

Чтобы создать окно, которое работает на нескольких платформах, а также использовать OpenGL ES контекст или кросс-платформенный ввод пользователя мы будем использовать пакет [winit](https://crates.io/crates/winit).

## Библиотеки Rust

Библиотеки Rust называются `crates`, а консольный менеджер пакетов для Rust называется `cargo`. Библиотеки могут быть найдены на центральном репозитории по ссылке [crates.io](https://crates.io).

Новуй пакет для Rust можно создать в используя команду `cargo new library-name`. Структура каталогов для новой пакета будет выглядеть очень похожэ на наш первый проект `hello-world`:

```txt
library-name
    src
        lib.rs
    Cargo.toml
```

Разница от исполняемого проекта, в том что в директории `src` вместо `main.rs` присутствует `lib.rs`.

Файл `Cargo.toml` описывает наш проект, в независимоти от того библиотека это, или исполняемый проект. Он содержит идентификатор преокта, список зависимостей, ссылку на документацию и многое другое. За более детальной информацией по содержимому манифеста `Cargo.toml` обратитесь [к документации по Cargo](https://doc.rust-lang.org/cargo/).

## Пакет winit

Пакет `winit` это кросс-платформенная библиотека для создания окна и обработки событий пользователя, таких как ввод пользователя с клавиатуры или мышки.

Можно посмотерть более детальную информацию об этом пакете по ссылке [winit](https://crates.io/crates/winit).


## Зависимости

Для того чтобы быстро довавлять зависимости в наш проект мы будем использовать `cargo-edit`. Для этого нужно его установить с помошью команды в вашем терминале:

```txt
> cargo install cargo-edit
```

Давайте создадим новый проект по имени `game`.

```txt
> cargo new --bin game
```

Чтобы добавить пакет `winit` в наш проект введите команду в терминале из корня проекта:

```txt
> cargo add winit
```

Эта команда добавит секцию `[dependencies]` в файл `Cargo.toml`, с пакетом `winit` в качестве зависимости:

_Cargo.toml, incomplete_

```toml
[dependencies]
winit = "0.25.0"
```

На момент написания этой главы новейшей версий `winit` является `0.25.0`. Вы можете ввести эту версию, чтобы убедиться, что все компилируется.

Указание зависимости позволяет загрузить её.

Чуть ниже я объясню назначение остальных зависимостей, сейчас же приведите секцию `[dependencies]` к следующему виду: 

```toml
[dependencies]
env_logger = "0.8"
khronos-egl = "4.1"
log = "0.4"
opengles = "0.1"
raw-window-handle = "0.3"
winit = "0.25"
```

Чтобы использовать заши зависимости, мы должны ссылаться на них.

## Использование winit

Укажите ссылки на неообходимые нам структуры, добавив в верхнюю часть файла `main.rs` соответствующий код:

```rust
use winit::{
    dpi::{LogicalSize, PhysicalSize, Size},
    event::{Event, KeyboardInput, VirtualKeyCode, WindowEvent},
    event_loop::{ControlFlow, EventLoop},
    window::WindowBuilder,
};
```

Этот позволит использовать короткие имена структур из пакета `winit`. Функции `winit` могут быть вызваны как `EventLoop::new()`.

## Инициализация winit.

Приведите код функции `main` к виду показаному ниже:

```rust
fn main() {
    let event_loop = EventLoop::new();

    let wb = WindowBuilder::new()
        .with_min_inner_size(Size::Logical(LogicalSize::new(64.0, 64.0)))
        .with_inner_size(Size::Physical(PhysicalSize::new(900, 700)))
        .with_title("Game".to_string());

    let _window = wb.build(&event_loop).unwrap();

    event_loop.run(move |event, _, control_flow| {
        *control_flow = ControlFlow::Wait;

        match event {
            Event::WindowEvent { event, .. } => match event {
                WindowEvent::CloseRequested => *control_flow = ControlFlow::Exit,
                WindowEvent::KeyboardInput {
                    input:
                        KeyboardInput {
                            virtual_keycode: Some(VirtualKeyCode::Escape),
                            ..
                        },
                    ..
                } => *control_flow = ControlFlow::Exit,
                WindowEvent::Resized(_) => {
                    // make changes based on window size
                }
                _ => {}
            },
            Event::RedrawEventsCleared => {
                // render window contents here
            }
            _ => {}
        }
    });
}
```

Ниже мы обсудим каждую деталь кода. Но сначала запустим!

```txt
> cargo run
```

Возможно вы не сразу заметите созданное окно, потому что мы ничего не отображали в нем.

```txt
   Compiling lesson-01-window v0.1.0 (/home/opengles-tutorial)
    Finished dev [unoptimized + debuginfo] target(s) in 9.31s
     Running `/home/opengles-tutorial/target/debug/lesson-01-window`
```

Прервите исполнение с помощью комбинации клавишь `Ctrl+C`.

## Разбор кода

Вернемся к коду. Давайте разберем его по крупицам.

```rust
let event_loop = EventLoop::new();
```

Эта строчка инициализирует цикл событий для обработки событий вашей оконной системы, таких как запрос на перерисовку окна или ввод пользователя. До тех пор пока работает цикл обработки событий работает и ваше приложение.

```rust
let wb = WindowBuilder::new()
    .with_min_inner_size(Size::Logical(LogicalSize::new(64.0, 64.0)))
    .with_inner_size(Size::Physical(PhysicalSize::new(900, 700)))
    .with_title("Game".to_string());
```
Строчки выше конфигурируют окно используя Builder паттерн. Мы задаем минимальный рамер окна, текущий размер окна и заголовок.

```rust
let _window = wb.build(&event_loop).unwrap();
```

Мы создаем окно, привязав его к циклу обработки событий оконной системы. Мы использовали имя переменной с префиксом поддеркивания `_window`, чтобы избежать предупреждающих сообщений компилятора, потому как мы еще не используем методы окна.

```rust
event_loop.run(move |event, _, control_flow| {
    ...
});
```

Конструкция выше запускает основной цикл обработки событий оконной системы. Программа работает до тех пор пока мы находимся в этом цикле. Единственным аргументом этого цикла является обработчик событий в виде замыкания.

Рассмотрим процесс обработки событий более детально. 
Обработчик событий помимо непосредственно события получает ссылку на объект, который контролирует цикл обработки событий.

```rust
*control_flow = ControlFlow::Wait;
```

Установив его в значение `ControlFlow::Wait` мы приостанавливаем цикл обработки событий, если события недоступны для обработки. Это идеально подходит для неигровых приложений, которые обновляются только в ответ на ввод пользователя и потребляют значительно меньше энергии/времени процессора, чем `ControlFlow::Poll`. 

Непосредственная обработка события подразумевает собой разбор перечисления события:

```rust
match event {
    Event::WindowEvent { event, .. } => match event {
        ...
    },
    Event::RedrawEventsCleared => {
        // render window contents here
    }
    _ => {}
}
```

В случае `Event::WindowEvent` мы обрабатываем события оконного менеджера и ввод пользователя.
Когда мы получаем `Event::RedrawEventsCleared`, то мы должны отрисовать содержимое окна. Здесь мы будем рендерить нашу графику.

Нам осталось более детально рассмотреть обработку событий окна.

```rust
Event::WindowEvent { event, .. } => match event {
    WindowEvent::CloseRequested => *control_flow = ControlFlow::Exit,
    WindowEvent::KeyboardInput {
        input:
            KeyboardInput {
                virtual_keycode: Some(VirtualKeyCode::Escape),
                ..
            },
        ..
    } => *control_flow = ControlFlow::Exit,
    WindowEvent::Resized(_) => {
        // make changes based on window size
    }
    _ => {}
},
```

`WindowEvent::CloseRequested` происходит в случае закрытия окна с помощью `x` кнопки окна. Логичной реакцией на это действие - будет завершение работы программы. Чтобы это сделать мы присвоим переменной состояния цикла обработки событий значение `ControlFlow::Exit`.

Обработка событий клавиатуры происходит по событию `WindowEvent::KeyboardInput`. В этой ситуации если пользователь нажмет клавишу `Esc`, мы также завершим работу программы.

Последнее событие котрое мы будем обрабатывать - `WindowEvent::Resized`. Оно отвечает за изменение размеров окна. И в этой ситуации я предпологаю что мы должны поменять внутренние переменные нашего приложения для корректного отображения нашей графики.

Остальные события на данном этапе мы не обрабатываем. Чтобы получить более детальную информацию по доступным событиям обратитесь к документации [winit](https://docs.rs/winit/)

Код этой главы доступен в основном [репозитории книги](https://github.com/angular-rust/opengles-tutorial/tree/main/lesson-01).
 
Теперь наше окно может быть окончательно закрыто, и [мы можем начать рисовать внутри окна](02-opengles-context.html).