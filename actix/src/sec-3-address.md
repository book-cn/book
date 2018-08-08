# 地址

参与者仅通过交换消息进行通信。发送方参与者可以选择<!--
-->等待该响应。不能直接引用参与者，只能通过其地址来引用。

有几种方式来获取参与者的地址。`Actor` trait 提供了<!--
-->两个辅助方法来启动参与者。这两个方法都会返回所启动参与者的地址。

以下是一个 `Actor::start()` 方法用法的示例。在这个示例中 `MyActor` 参与者是<!--
-->异步的，并且在与调用者相同的线程中启动。

```rust
# extern crate actix;
# use actix::prelude::*;
struct MyActor;
impl Actor for MyActor {
    type Context = Context<Self>;
}

# fn main() {
# System::new("test");
let addr = MyActor.start();
# }
```

异步参与者可以由 `Context` 对象获取其地址。该上下文需要实现
`AsyncContext` trait。`AsyncContext::address()` 提供了参与者的地址。

```rust
# extern crate actix;
# use actix::prelude::*;
struct MyActor;
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       let addr = ctx.address();
    }
}
# fn main() {}
```

## 信箱

所有消息都会首先到达参与者的信箱，然后该参与者执行上下文<!--
-->会调用指定的消息处理程序。信箱通常是有限的。其容量<!--
-->由上下文实现指定。对于 `Context` 类型，
默认容量设置为 16 条消息，可以通过
[*Context::set_mailbox_capacity()*](../actix/struct.Context.html#method.set_mailbox_capacity) 扩容。

## 消息

为了能够处理指定消息，参与者必须提供<!--
-->这种消息的 [`Handler<M>`](../actix/trait.Handler.html) 实现。
所有消息都是静态类型的。可以使用异步方式处理消息<!--
-->。参与者可以产生其他参与者或者将 future 或
stream 添加到执行上下文。参与者 trait 提供了几种可以<!--
-->控制参与者生命周期的方法。

如需向参与者发送消息，需要使用 `Addr` 对象。`Addr` 提供了几种<!--
-->发送消息的方式。

  * `Addr::do_send(M)`——这个方法会忽略参与者的信箱容量，并且<!--
  -->无条件将该消息放入信箱。这个方法不会返回<!--
  -->消息处理的结果，并且如果参与者消失就会静默失败。

  * `Addr::try_send(M)`——这个方法会立即尝试发送该消息。如果<!--
  -->信箱已满或者关闭（参与者已死），那么这个方法返回
  [`SendError`](../actix/prelude/enum.SendError.html)。

  * `Addr::send(M)`——这个消息返回一个可解析出消息处理过程的<!--
  -->结果的 future 对象。如果返回的 `Future` 对象被 drop，那么<!--
  -->会取消该消息。

## 收信方

收信方是仅支持一种类型消息的一种地址的专用版。
可以用于需要将消息发送给不同类型的参与者的场景。
可以用 `Addr::recipient()` 由地址创建收信方对象。

例如，收信方可以用于订阅系统。在以下示例中，
`ProcessSignals` 参与者向所有订阅者发送 `Signal` 消息。订阅者可以<!--
-->是实现了 `Handler<Signal>` trait 的任何参与者。

```rust
# #[macro_use] extern crate actix;
# use actix::prelude::*;
#[derive(Message)]
struct Signal(usize);

/// 订阅处理信号。
#[derive(Message)]
struct Subscribe(pub Recipient<Signal>);

/// 提供信号订阅的参与者
struct ProcessSignals {
    subscribers: Vec<Recipient<Signal>>,
}

impl Actor for ProcessSignals {
    type Context = Context<Self>;
}

impl ProcessSignals {

    /// 将信号发送给所有订阅者
    fn send_signal(&mut self, sig: usize) {
        for subscr in &self.subscribers {
           subscr.do_send(Signal(sig));
        }
    }
}

/// 订阅信号
impl Handler<Subscribe> for ProcessSignals {
    type Result = ();

    fn handle(&mut self, msg: Subscribe, _: &mut Self::Context) {
        self.subscribers.push(msg.0);
    }
}
# fn main() {}
```
