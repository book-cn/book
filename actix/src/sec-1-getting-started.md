# 入门

让我们来创建并运行第一个 actix 应用程序。我们会创建一个新的依赖于 actix 的 Cargo
项目，然后运行该应用程序。

在上一节中，我们已经安装了所需的 rust 版本。现在来创建新的 cargo 项目。

## Ping 参与者

我们来写第一个 actix 应用程序吧！首先创建一个新的基于二进制的
Cargo 项目并切换到新目录中：

```bash
cargo new actor-ping
cd actor-ping
```

现在，将 actix 添加为项目的依赖，即确保 Cargo.toml
中包含以下内容：

```toml
[dependencies]
actix = "0.10.0-alpha.3"
actix-rt = "1.1" # <-- Runtime for actix
```

我们来创建一个接受 `Ping` 消息并以 ping 处理后的数字作为响应的参与者。

参与者（actor）是实现 `Actor` trait 的类型：

```rust
# extern crate actix;
use actix::prelude::*;

struct MyActor {
    count: usize,
}

impl Actor for MyActor {
    type Context = Context<Self>;
}
#
# fn main() {}
```

每个参与者都有一个执行上下文，对于 `MyActor` 我们会使用 `Context<A>`。关于<!--
-->参与者上下文的更多信息在下一节中介绍。

现在需要定义参与者需要接受的消息（`Message`）。消息可以是实现
`Message` trait 的任何类型。

```rust
# extern crate actix;
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "usize")]
struct Ping(usize);
#
# fn main() {}
```

`Message` trait 的主要目的是定义结果类型。`Ping` 消息定义了
`usize`，表示任何可以接受 `Ping` 消息的参与者都需要<!--
-->返回 `usize` 值。

最后，需要声明我们的参与者 `MyActor` 可以接受 `Ping` 并处理它。
为此，该参与者需要实现 `Handler<Ping>` trait。

```rust
# extern crate actix;
# use actix::prelude::*;
#
# struct MyActor {
#    count: usize,
# }
# impl Actor for MyActor {
#     type Context = Context<Self>;
# }
#
# struct Ping(usize);
#
# impl Message for Ping {
#    type Result = usize;
# }
#
impl Handler<Ping> for MyActor {
    type Result = usize;

    fn handle(&mut self, msg: Ping, _ctx: &mut Context<Self>) -> Self::Result {
        self.count += msg.0;

        self.count
    }
}
#
# fn main() {}
```

就是这样。现在只需要启动我们的参与者并向其发送消息。
启动过程取决于参与者的上下文实现。在本例中我们可以使用<!--
-->基于 tokio/future 的 `Context<A>`。可以用 `Actor::start()`
或者 `Actor::create()` 来启动。前者用于可以立即创建参与者实例的场景。
后者用于在创建参与者实例之前需要访问上下文对象的场景<!--
-->。对于 `MyActor` 参与者，我们可以使用 `start()`。

所有与参与者的通信都通过地址进行。可以用 `do_send` 发送一条消息<!--
-->而不等待响应，也可以向一个参与者用 `send` 发送指定消息。
`start()` 与 `create()` 都会返回一个地址对象。

在以下示例中，我们会创建一个 `MyActor` 参与者并发送一条消息。

Here we use the actix-rt as way to start our System and drive our main Future
so we can easily `.await` for the messages sent to the Actor.

```rust
# extern crate actix;
# extern crate actix_rt;
# use actix::prelude::*;
# struct MyActor {
#    count: usize,
# }
# impl Actor for MyActor {
#     type Context = Context<Self>;
# }
#
# struct Ping(usize);
#
# impl Message for Ping {
#    type Result = usize;
# }
# impl Handler<Ping> for MyActor {
#     type Result = usize;
#
#     fn handle(&mut self, msg: Ping, ctx: &mut Context<Self>) -> Self::Result {
#         self.count += msg.0;
#         self.count
#     }
# }
#
#[actix_rt::main] 
async fn main() {
    // 启动新的参与者
    let addr = MyActor { count: 10 }.start();

    // 发送消息并获取结果 future
    let res = addr.send(Ping(10)).await;

    // handle() returns tokio handle
    println!("RESULT: {}", res.unwrap() == 20);

    // stop system and exit
    System::current().stop();
}
```

`#[actix_rt::main]` starts the system and block until future resolves.

Ping 示例可在[示例目录](https://github.com/actix/actix/tree/master/examples/)中找到。
