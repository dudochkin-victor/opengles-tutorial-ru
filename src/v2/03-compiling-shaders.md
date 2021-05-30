# Компиляция шейдеров

> [Ранее](02-opengles-context.html), мы подключили OpenGL ES контекст к окну, чтобы обрабатывать события от пользователя и выводить графику.

В этой главе, мы будем работать над рендерингом классического треугольника OpenGL. Классический, потому что каждый учебник OpenGL делает это.

Но сначала мы узнаем, как создавать безопасные абстракции в Rust, и создадим инструменты для компиляции шейдера и компоновки программы. 

## "Современный" OpenGL

Мы будем использовать то, что называется "современным OpenGL". Оказывается, давным-давно это было не так современно. Мы не будем обсуждать здесь графический конвейер с самого начала: вместо этого я предлагаю вам воспользоваться другим "современным учебником OpenGL". Я буду использовать [это замечательное руководство в качестве основы](https://learnopengl.com/).
Эта глава будет посвящена реализации на Rust урока ["Hello Triangle"](https://learnopengl.com/Getting-started/Hello-Triangle).

## Дополнительная настройка контекста OpenGL 

В [пердыдущей главе](02-opengles-context.html) мы говорили о создании окна. В основном мы так и поступали, за исключением того, что я забыл кое-что.

Во-первых, нам нужно указать минимальную версию OpenGL для использования. В нашем случае это можно сделать с помощью атрибутов EGL контекста: 

```rust
    let ctx_attribs = [egl::NONE];
    let ctx = egl
        .create_context(display, config, None, &ctx_attribs)
        .expect("unable to create EGL context");
```

Как вы можете видеть, мы задали пустой список аттрибутов EGL контекста. Если не настраивать аттрибуты контекста, то графическая подсистема возьмет минимальную версию OpenGL ES. В моем случае это OpenGL ES 1.1.

Чтобы увидеть как это работает, добавьте несколько строк после вызова команды `egl.make_current`:

```rust
println!(
    "GL_RENDERER = {}",
    gl::get_string(gl::GL_RENDERER).unwrap_or("Unknown".into())
);
println!(
    "GL_VERSION = {}",
    gl::get_string(gl::GL_VERSION).unwrap_or("Unknown".into())
);
println!(
    "GL_VENDOR = {}",
    gl::get_string(gl::GL_VENDOR).unwrap_or("Unknown".into())
);
println!(
    "GL_EXTENSIONS = {}",
    gl::get_string(gl::GL_EXTENSIONS).unwrap_or("Unknown".into())
);
```

В моем случае это выглядит вот так:

```txt
> cargo run
   Compiling lesson-02-opengl-context v0.1.0 (/home/opengles-tutorial)
    Finished dev [unoptimized + debuginfo] target(s) in 7.27s
     Running `/home/opengles-tutorial/target/debug/lesson-02-opengl-context`
GL_RENDERER = GeForce GTX 1050/PCIe/SSE2
GL_VERSION = OpenGL ES 1.1 NVIDIA 460.67
GL_VENDOR = NVIDIA Corporation
GL_EXTENSIONS = GL_EXT_debug_label GL_EXT_map_buffer_range GL_EXT_robustness 
GL_EXT_texture_compression_dxt1 GL_EXT_texture_compression_s3tc 
GL_EXT_texture_format_BGRA8888 GL_KHR_debug GL_EXT_memory_object 
GL_EXT_memory_object_fd GL_NV_memory_object_sparse GL_EXT_semaphore 
GL_EXT_semaphore_fd GL_NV_timeline_semaphore GL_NV_memory_attachment 
GL_NV_texture_compression_s3tc GL_OES_compressed_ETC1_RGB8_texture 
GL_EXT_compressed_ETC1_RGB8_sub_texture GL_OES_compressed_paletted_texture 
GL_OES_draw_texture GL_OES_EGL_image GL_OES_EGL_image_external GL_OES_EGL_sync 
GL_OES_element_index_uint GL_OES_extended_matrix_palette GL_OES_fbo_render_mipmap 
GL_OES_framebuffer_object GL_OES_matrix_get GL_OES_matrix_palette 
GL_OES_packed_depth_stencil GL_OES_point_size_array GL_OES_point_sprite 
GL_OES_rgb8_rgba8 GL_OES_read_format GL_OES_stencil8 GL_OES_texture_cube_map 
GL_OES_texture_npot GL_OES_vertex_half_float
```

Эта книга нацелена на изучение OpenGL ES версии 2.0 и выше. Поэтому настроим аттрибуты контекста, изменив всего одну строчку:

```rust
let ctx_attribs = [egl::CONTEXT_CLIENT_VERSION, 2, egl::NONE];
```

Теперь вывод будет совершенно иным:

```txt
В моем случае это выглядит вот так:

```txt
> cargo run
   Compiling lesson-02-opengl-context v0.1.0 (/home/opengles-tutorial)
    Finished dev [unoptimized + debuginfo] target(s) in 7.27s
     Running `/home/opengles-tutorial/target/debug/lesson-02-opengl-context`
GL_RENDERER = GeForce GTX 1050/PCIe/SSE2
GL_VERSION = OpenGL ES 3.2 NVIDIA 460.67
...
```

Помимо того, что теперь мы можем работать с OpenGL ES 3.2, также изменились поддерживаемые OpenGL расширения и их количество. Пока что примите OpenGL расширения как данность, мы обсудим их в следующих главах.

Во-вторых, нам нужно настроить область просмотра. Мы можем сделать это один раз, непосредственно перед первым вызовом `gl::clear_color`: 


```rust
gl::viewport(0, 0, 900, 700);
```

## Шейдеры

Мы создадим вспомогательную функцию для компиляции шейдера из строки, а затем другую функцию для связывания скомпилированных шейдеров с программой. 

Сначала попробуйте добавить этот код в перед функцийе `main` в файл `main.rs`:

```rust
fn shader_from_source(source: &str) -> u32 {
    // continue here
}
```

Учитывая этот код, функция `shader_from_source` должна возвращать идентификатор шейдера. 

Однако создание шейдера может завершиться неудачно, и мы можем захотеть получить сообщение об ошибке.
Для этого мы изменим тип возвращаемого значения на `Result<u32, String>`. Если компиляция шедера завершится удачно, то мы получим идентификатор шейдера, иначе мы сможем извлечь сообщение об ошибке.

Начнем с получения id объекта шейдера:

```rust
fn shader_from_source(source: &str) -> Result<u32, String> {
    let id = gl::create_shader(gl::GL_VERTEX_SHADER);

    // continue here
}
```

Ха! Нам нужно указать тип шейдера. Давайте улучшим сигнатуру функции и добавим к ней тип шейдера. `type` - зарезервированное ключевое слово в Rust, но мы можем назвать его` kind`: 

```rust
fn shader_from_source(
    source: &str,
    kind: kind: gl::GLenum
) -> Result<Result<u32, String>> {
    let id = gl::create_shader(kind);

    // continue here
}
```

Затем нам нужно установить источник для объекта шейдера, используя обертку к [glShaderSource](http://docs.gl/gl4/glShaderSource) функции. 
Мы можем установить источник шейдера и скомпилировать его: 

```rust
    gl::shader_source(id, source.as_bytes());
    gl::compile_shader(id);

// continue here
```

Мы могли бы вернуть `Ok(id)` сейчас и двигаться дальше, однако нам __реально__ нужно видеть правильное сообщение об ошибке, если шейдер не компилируется. 

Таким образом, получаем статус компиляции шейдера: 

```rust
let success = gl::get_shaderiv(id, gl::GL_COMPILE_STATUS);
```

И если это `0`, мы вернем строку с ошибкой, иначе `Ok(id)`: 

```rust
if success == 0 {
    // continue here
}

Ok(id)
```

Нам нужно будет записать возвращенную ошибку в буфер, поэтому нам нужно знать требуемую длину этого буфера. 

Мы получим `len` запросив `GL_INFO_LOG_LENGTH` для объекта шейдера: 

```rust
let len = gl::get_shaderiv(id, gl::GL_INFO_LOG_LENGTH);

// continue here
```

При этом мы можем попросить OpenGL записать журнал информации о шейдере в значение нашей ошибки и наконец, мы можем вернуть ошибку : 

```rust
return match gl::get_shader_info_log(id, len) {
    Some(message) => Err(message),
    None => Ok(id)
};

// continue here
```
Финальный код функции `shader_from_source`:

```rust
fn shader_from_source(source: &str, kind: gl::GLenum) -> Result<u32, String> {
    let id = gl::create_shader(kind);

    gl::shader_source(id, source.as_bytes());
    gl::compile_shader(id);

    let success = gl::get_shaderiv(id, gl::GL_COMPILE_STATUS);

    if success == 0 {
        let len = gl::get_shaderiv(id, gl::GL_INFO_LOG_LENGTH);

        return match gl::get_shader_info_log(id, len) {
            Some(message) => Err(message),
            None => Ok(id)
        };
    }

    Ok(id)
}
```

## Программа

Программа связывания требует следующих вызовов OpenGL:

```rust
let program_id = gl::create_program();

gl::attach_shader(program_id, vert_shader);
gl::attach_shader(program_id, frag_shader);

gl::link_program(program_id);
```

По аналогии с компиляцмей шейдеров напишем функцию для программы связывания.
Вместо `get_shaderiv`, мы испольуем `get_programiv`, instead of `get_shader_info_log` we use `get_program_info_log`.

```rust
fn program_from_shaders(vert_shader: u32, frag_shader: u32) -> Result<u32, String> {
    let program_id = gl::create_program();

    gl::attach_shader(program_id, vert_shader);
    gl::attach_shader(program_id, frag_shader);

    gl::link_program(program_id);

    // error handling here
    let success = gl::get_programiv(program_id, gl::GL_LINK_STATUS);

    if success == 0 {
        let len = gl::get_programiv(program_id, gl::GL_INFO_LOG_LENGTH);     

        return match gl::get_program_info_log(program_id, len) {
            Some(message) => Err(message),
            None => Ok(program_id)
        };
    }

    gl::detach_shader(program_id, vert_shader);
    gl::detach_shader(program_id, frag_shader);

    Ok(program_id)
}
```

Одно маленькое уточнение: `gl::delete_shader` не удалит шейдер, если он все еще прикреплен к программе. Поэтому мы отключаем шейдер после его связывания.

## Использование наших шейдеров и программ связывания 

Для начала добавим код шейдеров в файл `main.rs`. Я добавил его перед функцией `shader_from_source`:

```rust
pub const VERT_SHADER: &str = r"
    attribute vec3 Position;

    void main()
    {
        gl_Position = vec4(Position, 1.0);
    }
";

pub const FRAG_SHADER: &str = r"
    precision mediump float;
    void main()
    {
        gl_FragColor = vec4(1.0, 0.5, 0.2, 1.0);
    }
";
```

Прямо перед основным циклом, мы можем скомпилировать наши вершинные и шейдерные программы: 

(main.rs, before loop)

```rust
let vertex_shader = match shader_from_source(VERT_SHADER, gl::GL_VERTEX_SHADER) {
    Ok(id) => {
        println!("Vertex Shader Compiled");
        id
    },
    Err(err) => panic!("{}", err)
};

let fragment_shader = match shader_from_source(FRAG_SHADER, gl::GL_FRAGMENT_SHADER) {
    Ok(id) => {
        println!("Fragment Shader Compiled");
        id
    },
    Err(err) => panic!("Error: {}", err)
};

// continue here
```

Затем мы связываем наши шейдеры: 

```rust
match program_from_shaders(vertex_shader, fragment_shader) {
    Ok(_) => println!("Program linked"),
    Err(err) => panic!("Error: {}", err)
}

// continue here
```

Запустим программу на выполнение:

```txt
> cargo run
   Compiling lesson-02-opengl-context v0.1.0 (/home/opengles-tutorial)
    Finished dev [unoptimized + debuginfo] target(s) in 7.27s
     Running `/home/opengles-tutorial/target/debug/lesson-02-opengl-context`
GL_RENDERER = GeForce GTX 1050/PCIe/SSE2
GL_VERSION = OpenGL ES 1.1 NVIDIA 460.67
GL_VENDOR = NVIDIA Corporation
GL_EXTENSIONS = GL_EXT_debug_label GL_EXT_map_buffer_range GL_EXT_robustness ...

Vertex Shader Compiled
Fragment Shader Compiled
Program linked
```

Однако мы пока ничего не увидим на экране, потому что мы еще не отправляем никаких команд рисования в OpenGL. Однако если вывод в терминале должен показать нам что шейдеры скомпилировались и программа слинкована.

Мы исправим это [в следующий раз](04-triangle.html).

Как всегда, [код этой части главы на github](https://github.com/angular-rust/opengles-tutorial/tree/main/lesson-03).
