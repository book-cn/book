# 参与者

Actix 是一个 rust 库，为开发并发应用程序提供了框架。

Actix 建立在[参与者模型（Actor model）](https://zh.wikipedia.org/wiki/%E5%8F%83%E8%88%87%E8%80%85%E6%A8%A1%E5%BC%8F)之上，它<!--
-->使应用程序可以编写为一组独立执行而又相互协作的
“参与者”，这些参与者通过消息进行通信。参与者是封装<!--
-->状态与行为并且会在由 actix 库提供的 *Actor System* 中运行的对象。

参与者在指定的上下文 [*Context<A>*](../actix/struct.Context.html) 中运行。
该上下文对象只在执行期间可用。每个参与者都有一个独立的<!--
-->执行上下文。该执行上下文还控制参与者的生命周期。

参与者仅通过交换消息进行通信。发送方参与者可以<!--
-->选择等待该响应。不能直接引用参与者，而要通过<!--
-->地址来引用。

任何 rust 类型都可以是一个参与者，只需实现 [*Actor*](../actix/trait.Actor.html) trait 即可。

为了能够处理指定消息，参与者必须提供<!--
-->这种消息的 [*Handler<M>*](../actix/trait.Handler.html) 实现。所有消息<!--
-->都是静态类型的。可以使用异步方式处理消息。
参与者可以产生其他参与者或者将 future 或 stream 添加到执行上下文。
`Actor` trait 提供了几种可以控制参与者生命周期的方法。


## 参与者生命周期

### 已启动（Started）

参与者总是以 `Started` 状态启动。在这一状态期间调用了该参与者的 `started()`
方法。`Actor` trait 为这个方法提供了默认实现。
在这一状态期间可以使用参与者上下文，并且该参与者可以启动更多参与者或者注册<!--
-->异步流或者做任何其他所需的配置操作。

### 运行中（Running）

调用参与者的 `started()` 方法后，该参与者会转换为 `Running` 状态。
参与者可以无限期地处于 `running` 状态。

### 停止中（Stopping）

在以下情况下，参与者的执行状态会变更为 `stopping` 状态：

* 该参与者自身调用了 `Context::stop`
* 该参与者的所有地址都已删除。即没有其他参与者引用它。
* 在上下文中没有注册事件对象。

一个参与者可以由 `stopping` 状态恢复为 `running` 状态，通过创建一个新的<!--
-->地址或者添加事件对象，以及通过返回 `Running::Continue` 实现。

如果一个参与者状态变更为 `stopping` 是因为调用了 `Context::stop()`，
那么该上下文会立即停止处理接入的消息，并调用
`Actor::stopping()`。如果参与者没有恢复到 `running` 状态，那么<!--
-->删除所有未处理的消息。

默认这个方法返回 `Running::Stop`，确认停止操作。

### 已停止（Stopped）

如果参与者在停止中状态期间没有修改执行上下文，那么参与者状态会变更<!--
-->为 `Stopped`。这个状态被认为是最终状态，此时该参与者会被 drop。


## 消息

一个 Actor 通过发送消息与其他参与者通信。在 actix 中的所有<!--
-->消息都是类型化的。消息可以是任何实现了
[Message](../actix/trait.Message.html) trait 的 rust 类型。*Message::Result* 定义了其返回值类型。
让我们来定义一个简单的 `Ping` 消息——接受这种消息的参与者需要返回
`io::Result<bool>`。

```rust
# extern crate actix;
use std::io;
use actix::prelude::*;

struct Ping;

impl Message for Ping {
    type Result = Result<bool, io::Error>;
}

# fn main() {}
```

## 产生一个参与者

如何启动一个参与者取决于它的上下文。可通过
[Actor](../actix/trait.Actor.html) trait 的
`start` 与 `create` 实现产生一个新的异步参与者。Actor trait 提供了几种不同的<!--
-->创建参与者的方式；更详细信息请查看其文档。

## 完整示例

```rust
# extern crate actix;
# extern crate futures;
use std::io;
use actix::prelude::*;
use futures::Future;

/// 定义消息
struct Ping;

impl Message for Ping {
    type Result = Result<bool, io::Error>;
}


// 定义参与者
struct MyActor;

// 为我们的参与者提供 Actor 实现
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       println!("Actor is alive");
    }

    fn stopped(&mut self, ctx: &mut Context<Self>) {
       println!("Actor is stopped");
    }
}

/// 为 `Ping` 消息定义处理程序
impl Handler<Ping> for MyActor {
    type Result = Result<bool, io::Error>;

    fn handle(&mut self, msg: Ping, ctx: &mut Context<Self>) -> Self::Result {
        println!("Ping received");

        Ok(true)
    }
}

fn main() {
    let sys = System::new("example");

    // 在当前线程启动 MyActor
    let addr = MyActor.start();

    // 发送 Ping 消息。
    // send() 消息返回 Future 对象，可解析出消息结果
    let result = addr.send(Ping);

    // 产生 future 给响应器
    Arbiter::spawn(
        result.map(|res| {
            match res {
                Ok(result) => println!("Got result: {}", result),
                Err(err) => println!("Got error: {}", err),
            }
#           System::current().stop();
        })
        .map_err(|e| {
            println!("Actor is probably dead: {}", e);
        }));

    sys.run();
}
```

## 以 MessageResponse 进行响应

我们来看看上例中为 `impl Handler` 定义的 `Result` 类型。看下我们是如何返回一个 `Result<bool, io::Error>` 的？我们能够以这种类型响应该参与者的接入消息，是因为它已经为该类型实现了 `MessageResponse` trait。这是该 trait 的定义：

```
pub trait MessageResponse<A: Actor, M: Message> {
    fn handle<R: ResponseChannel<M>>(self, ctx: &mut A::Context, tx: Option<R>);
}
```

有时会需要以没有为其实现这个 trait 的类型来响应接入的消息。当出现这种情况时，我们可以自己实现该 trait。以下是一个示例，其中我们以 `GotPing` 响应 `Ping` 消息，并且以 `GotPong` 响应 `Pong` 消息。

```rust
# extern crate actix;
# extern crate futures;
use actix::dev::{MessageResponse, ResponseChannel};
use actix::prelude::*;
use futures::Future;

enum Messages {
    Ping,
    Pong,
}

enum Responses {
    GotPing,
    GotPong,
}

impl<A, M> MessageResponse<A, M> for Responses
where
    A: Actor,
    M: Message<Result = Responses>,
{
    fn handle<R: ResponseChannel<M>>(self, _: &mut A::Context, tx: Option<R>) {
        if let Some(tx) = tx {
            tx.send(self);
        }
    }
}

impl Message for Messages {
    type Result = Responses;
}

// 定义参与者
struct MyActor;

// 为我们的参与者提供 Actor 实现
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
        println!("Actor is alive");
    }

    fn stopped(&mut self, ctx: &mut Context<Self>) {
        println!("Actor is stopped");
    }
}

/// 为 `Messages` 枚举定义处理程序
impl Handler<Messages> for MyActor {
    type Result = Responses;

    fn handle(&mut self, msg: Messages, ctx: &mut Context<Self>) -> Self::Result {
        match msg {
            Messages::Ping => Responses::GotPing,
            Messages::Pong => Responses::GotPong,
        }
    }
}

fn main() {
    let sys = System::new("example");

    // 在当前线程启动 MyActor
    let addr = MyActor.start();

    // 发送 Ping 消息。
    // send() 消息返回 Future 对象，可解析出消息结果
    let ping_future = addr.send(Messages::Ping);
    let pong_future = addr.send(Messages::Pong);

    // 将 pong_future 产生到事件循环上
    Arbiter::spawn(
        pong_future
            .map(|res| {
                match res {
                    Responses::GotPing => println!("Ping received"),
                    Responses::GotPong => println!("Pong received"),
                }
#               System::current().stop();
            })
            .map_err(|e| {
                println!("Actor is probably dead: {}", e);
            }),
    );

    // 将 ping_future 产生到事件循环上
    Arbiter::spawn(
        ping_future
            .map(|res| {
                match res {
                    Responses::GotPing => println!("Ping received"),
                    Responses::GotPong => println!("Pong received"),
                }
#               System::current().stop();
            })
            .map_err(|e| {
                println!("Actor is probably dead: {}", e);
            }),
    );

    sys.run();
}
```
