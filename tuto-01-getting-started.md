# 快速开始

## 创建项目

首先, 创建一个新项目:

```sh
cargo new --bin my_project
cd my_project
```

在你刚创建的目录下的 `Cargo.toml` 文件中含有项目的元数据, `src/main.rs` 文件包含了 Rust 的源代码. 如果在目录下的文件不是 `src/main.rs` 而是 `src/lib.rs` , 说明你忘了输入 `--bin` 选项; 只要将 `src/lib.rs` 重命名为 `src/main.rs` 就可以了.

在 `Cargo.toml` 文件中添加如下依赖, 以在项目中导入 glium 库:

```toml
[dependencies]
glium = "*"
```

在使用依赖库之前, 需要在 `src/main.rs` 中添加如下代码以导入库:

```rust
#[macro_use]
extern crate glium;

fn main() {
}
```

是时候开始填满 `main` 函数了!

## 创建窗口

创建图形应用的第一步就是创建一个窗口. 如果你熟悉 OpenGL 那一套, 你应该了解其中的复杂程度. 无论是创建窗口还是创建上下文(Context), 在不同平台下的你要写的代码是不一样的, 唯一不变的是: 它非常无聊. 幸运的是, 这正是 **glutin** 库所擅长的东西.

使用 glutin 初始化 OpenGL 窗口需要经过如下几步:

1. 创建 `EventsLoop<T>` 用于处理窗口和设备的事件.
2. 使用 `glium::glutin::WindowBuilder::new()` 指定窗口的参数. 这些参数与 OpenGL 无关, 而是窗口特有的属性.
3. 使用`glium::glutin::ContextBuilder::new()`指定上下文(Context)参数. 我们可以在这里设置 OpenGL 特定的属性, 例如垂直同步与多重采样.
4. 创建 OpenGL 窗口(在 glium 中称为 `Display`):  
   `glium::Display::new(window, context, &events_loop).unwrap()`  
   这句代码使用给定的窗口和上下文创建了一个 Dispaly, 并且使用所给的 events_loop 注册窗口.

```rust
fn main() {
    use glium::glutin;

    let mut events_loop = glutin::event_loop::EventsLoop::new();
    let window = glutin::WindowBuilder::new();
    let context = glutin::ContextBuilder::new();
    let display = glium::Display::new(window, context, &events_loop).unwrap();
}
```

但是有一个问题: 在窗口创建完成时, 我们的 main 函数就退出了, 导致 `display` 的析构函数关闭了窗口. 为了避免这个问题, 我们需要创建一个无限循环, 直到收到 `CloseRequested` 事件为止:

```rust
events_loop.run(move |ev, _, control_flow| {
        let next_frame_time = std::time::Instant::now() + std::time::Duration::from_nanos(16_666_667);
        *control_flow = glutin::event_loop::ControlFlow::WaitUntil(next_frame_time);
        match ev {
            glutin::event::Event::WindowEvent { event, .. } => match event {
                glutin::event::WindowEvent::CloseRequested => {
                    *control_flow = glutin::event_loop::ControlFlow::Exit;
                    return;
                }
                _ => return,
            },
            _ => (),
        }
    });
```

虽然, 这段代码将导致 CPU 占用率达到 100%, 但是已经解决了上面所说的问题. 在实际的应用程序中, 你应当使用垂直同步或者在每次循环结束时 sleep 几毫秒, 不过这是之后要考虑的事了.

你现在可以运行 `cargo run`. 在 Cargo 下载完 glium 及其依赖并编译好之后, 你就能看见一个还过得去的小窗口了.

### [增]开启垂直同步

上述程序会导致 CPU 占用达到 100%是因为没有限制窗口的渲染帧率, 导致程序尽力渲染尽可能高的帧数, 想要使占用率降低, 可以开启垂直同步限制渲染的帧数, 以节约 CPU 资源
具体做法:

```rust
let cb = glium::glutin::ContextBuilder::new().with_vsync(true);
```

在创建 Opengl Context(OpenGL 上下文)中设置 Vsync 开启即可

!注意!:垂直同步在没有渲染之前都是正常的, 在将渲染代码丢入事件循环后, 会使得整个窗口卡死, 在 AMD 的核显与 Nvidia GTX 系列显卡上都有同样的问题, 原生调用 Opengl 垂直同步不会出现此问题, 猜测是 glium 库对于 Opengl 垂直同步 API 包装有错误

### 清除颜色

然而, 窗口里的内容并不怎么吸引人. 它也许是一片空白, 或者是一张随机的图片, 也可能是一些雪花, 这取决于你使用什么系统. 我们需要在窗口内绘制图形, 因此系统并不需要将窗口内的颜色初始化为某个特定的值.

Glium 和 OpenGL 的 API 就像 Windows 下的画笔工具. 首先我们有一幅空白的画布, 我们可以在上面画一个又一个图形. 到你觉得满意为止. 和绘图软件不一样的是, 你不想你的用户看见绘画的过程. 他们能看见的应该只有最终的结果.

glium 使用 `Frame` 对象来实现这一机制. 如果你打算在窗口中绘制图形, 首先应该调用 `display.draw()` 方法来创建一个 `Frame`:

```rust
let mut target = display.draw();
```

之后, 我们就能将 `target` 作为画布(drawing surface). OpenGL 和 glium 提供的一个操作就是使用给定的颜色填充画布. 这正是我们要做的.

```rust
target.clear_color(0.0, 0.0, 1.0, 1.0);
```

注意: 我们需要先导入 `Surface` trait, 然后才能使用这个函数:

```rust
use glium::Surface;
```

我们传递给 `clear_color` 的四个值表示颜色的四个组成部分: 红, 绿, 蓝和透明度. 取值范围为[0.0, 1.0]. 这里我们选择的是不透明的蓝色.

正如我之前解释的, 现在用户没法在屏幕上看见蓝色. 如果我们是在写一个真正的应用的话, 我们也许会画上角色, 武器, 地面, 天空等等. 但是在这个指南中, 我们做到这一步就足够了:

```rust
target.finish().unwrap();
```

`finish()` 函数意味着我们结束了绘制. 它将销毁 `Frame` 对象, 并将画布复制到窗口上. 现在, 我们的窗口被一片蓝色填满了.

以下是所有的代码:

```rust
fn main() {
    use glium::{glutin, Surface};

    let mut events_loop = glium::glutin::event_loop::EventsLoop::new();
    let window = glium::glutin::WindowBuilder::new();
    let context = glium::glutin::ContextBuilder::new();
    let display = glium::Display::new(window, context, &events_loop).unwrap();

    // 事件循环
    events_loop.run(move |ev, _, control_flow| {
        // 对屏幕进行渲染
        let mut target = display.draw();
        target.clear_color(0.4, 0.5, 0.8, 0.8);
        target.finish().unwrap();

        // 这里为手动控制帧速率, 1s = 1000000000 nanos 1000000000 / 60 = 16666667 nanos每帧, 这里使用的是纳秒
        // 这里也可以使用ms(毫秒)来得到帧生成时间为16.666667, 但是更推荐更小一级的单位, 首先它的数据长度和浮点数相当
        // 其次在不支持硬件浮点加速的CPU上, 该代码也能获得不错的性能
        let next_frame_time = std::time::Instant::now() + std::time::Duration::from_nanos(16_666_667);
        *control_flow = glutin::event_loop::ControlFlow::WaitUntil(next_frame_time);

        match ev {
            glutin::event::Event::WindowEvent { event, .. } => match event {
                glutin::event::WindowEvent::CloseRequested => {
                    *control_flow = glutin::event_loop::ControlFlow::Exit;
                    return;
                }
                _ => return,
            },
            _ => (),
        }
    });
}
```

至此,你会得到一个具有纯色背景的窗口

> 原地址: <https://github.com/glium/glium/blob/master/book/tuto-01-getting-started.md>
