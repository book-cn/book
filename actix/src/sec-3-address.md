# 地址

参与者仅通过交换消息进行通信。发送方参与者可以选择<!--
-->等待该响应。不能直接引用参与者，只能通过其地址来引用。

有几种方式来获取参与者的地址。`Actor` trait 提供了<!--
-->两个辅助方法来启动参与者。这两个方法都会返回所启动参与者的地址。

以下是一个 `Actor::start()` 方法用法的示例。在这个示例中 `MyActor` 参与者是<!--
-->异步的，并且在与调用者相同的线程中启动——线程会在
[SyncArbiter] 章节中介绍。

```rust
# extern crate actix;
# use actix::prelude::*;
#
struct MyActor;
impl Actor for MyActor {
    type Context = Context<Self>;
}

# fn main() {
# System::new("test");
let addr = MyActor.start();
# }
```

异步参与者可以由 `Context` 结构获取其地址。该上下文需要实现
`AsyncContext` trait。`AsyncContext::address()` 提供了参与者的地址。

```rust
# extern crate actix;
# use actix::prelude::*;
#
struct MyActor;

impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       let addr = ctx.address();
    }
}
#
# fn main() {}
```

[SyncArbiter]: ./sec-6-sync-arbiter.md

## 消息

为了能够处理指定消息，参与者必须提供<!--
-->这种消息的 [`Handler<M>`] 实现。
所有消息都是静态类型的。可以使用异步方式处理消息<!--
-->。参与者可以产生其他参与者或者将 future 或
stream 添加到执行上下文。参与者 trait 提供了几种可以<!--
-->控制参与者生命周期的方法。

如需向参与者发送消息，需要使用 `Addr` 对象。`Addr` 提供了几种<!--
-->发送消息的方式。

  * `Addr::do_send(M)`——这个方法会忽略消息发送中的任何错误。如果信箱<!--
  -->已满，那么仍会绕过限制将该消息排入队列。如果该参与者的信箱已关闭，
  那么会以静默方式丢弃该消息。这个方法不会返回结果，因此<!--
  -->信箱关闭及发生故障都无从知悉。

  * `Addr::try_send(M)`——这个方法会立即尝试发送该消息。如果<!--
  -->信箱已满或者关闭（参与者已死），那么这个方法返回
  [`SendError`]。

  * `Addr::send(M)`——这个消息返回一个可解析出消息处理过程的<!--
  -->结果的 future 对象。如果返回的 `Future` 对象被 drop，那么<!--
  -->会取消该消息。

[`Handler<M>`]: https://actix.rs/actix/actix/trait.Handler.html
[`SendError`]: https://actix.rs/actix/actix/prelude/enum.SendError.html

## 收信方

收信方是仅支持一种类型消息的一种地址的专用版。
可以用于需要将消息发送给不同类型的参与者的场景。
可以用 `Addr::recipient()` 由地址创建收信方对象。

Address objects require an actor type, but if we just want to send a specific message 
to an actor that can handle the message, we can use the Recipient interface.

例如，收信方可以用于订阅系统。在以下示例中，
`OrderEvents` 参与者向所有订阅者发送 `OrderShipped` 消息。订阅者可以<!--
-->是实现了 `Handler<OrderShipped>` trait 的任何参与者。

```rust
# extern crate actix;
use actix::prelude::*;

#[derive(Message)]
#[rtype(result = "()")]
struct OrderShipped(usize);

#[derive(Message)]
#[rtype(result = "()")]
struct Ship(usize);

/// Subscribe to order shipped event.
#[derive(Message)]
#[rtype(result = "()")]
struct Subscribe(pub Recipient<OrderShipped>);

/// Actor that provides order shipped event subscriptions
struct OrderEvents {
    subscribers: Vec<Recipient<OrderShipped>>,
}

impl OrderEvents {
    fn new() -> Self {
        OrderEvents {
            subscribers: vec![]
        }
    }
}

impl Actor for OrderEvents {
    type Context = Context<Self>;
}

impl OrderEvents {
    /// Send event to all subscribers
    fn notify(&mut self, order_id: usize) {
        for subscr in &self.subscribers {
           subscr.do_send(OrderShipped(order_id));
        }
    }
}

/// Subscribe to shipment event
impl Handler<Subscribe> for OrderEvents {
    type Result = ();

    fn handle(&mut self, msg: Subscribe, _: &mut Self::Context) {
        self.subscribers.push(msg.0);
    }
}

/// Subscribe to ship message
impl Handler<Ship> for OrderEvents {
    type Result = ();
    fn handle(&mut self, msg: Ship, ctx: &mut Self::Context) -> Self::Result {
        self.notify(msg.0);
        System::current().stop();
    }

} 

/// Email Subscriber 
struct EmailSubscriber;
impl Actor for EmailSubscriber {
    type Context = Context<Self>;
}

impl Handler<OrderShipped> for EmailSubscriber {
    type Result = ();
    fn handle(&mut self, msg: OrderShipped, _ctx: &mut Self::Context) -> Self::Result {
        println!("Email sent for order {}", msg.0)
    }
    
}
struct SmsSubscriber;
impl Actor for SmsSubscriber {
    type Context = Context<Self>;
}

impl Handler<OrderShipped> for SmsSubscriber {
    type Result = ();
    fn handle(&mut self, msg: OrderShipped, _ctx: &mut Self::Context) -> Self::Result {
        println!("SMS sent for order {}", msg.0)
    }
    
}

fn main() {
    let system = System::new("events");
    let email_subscriber = Subscribe(EmailSubscriber{}.start().recipient());
    let sms_subscriber = Subscribe(SmsSubscriber{}.start().recipient());
    let order_event = OrderEvents::new().start();
    order_event.do_send(email_subscriber);
    order_event.do_send(sms_subscriber);
    order_event.do_send(Ship(1));
    system.run();
}
```
