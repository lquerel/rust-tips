# Async Patterns

## Detect blocking code 
TBD https://rickyhan.com/jekyll/update/2019/12/22/convert-to-async-rust.html 
 
## Mutex (std::sync vs tokio::sync)

A `std::Mutex` can be used with async. It's recommended to use it when the lock is never held across an await point.
More details on the differences between `std::Mutex` and `tokio::Mutex` [here](https://tokio-rs.github.io/tokio/doc/tokio/sync/struct.Mutex.html#which-kind-of-mutex-should-you-use).


## Communicate from a non-async function to an async function which was called from an async function

Call an async function from a non-async function (you don't control) that has been called from another async function is challenging. In this situation you can't 
call the Tokio 'block_on' or 'spawn_blocking' function inside the sync function because a Tokio runtime nesting issue (i.e. Cannot start a runtime from within a 
runtime).

One of the option to solve this issue is to use a Tokio channel and leverage the fact that a Tokio channel exposes methods working in "sync" mode 
(try_send and blocking_recv).

Inside your async application (before to call the non-async function/framework).
```rust
let channel_buffer_size = 1000;
let (sender, mut receiver) = tokio::sync::mpsc::channel::<Something>(channel_buffer_size);

let async_consumer = tokio::task::spawn(async move {
  // Do something with the data emitted by the receiver. 
});

// Store the sender somewhere accessible by the non-async function. 

// ...
tokio::join!((async_consumer);
```

Inside your non-async method (provided that self is containing the sender created previously) 
```rust
let result = self.sender.try_send(<your something>);

if let Err(err) = result {
  // Do something in case the channel is full or no longer open
}
```

Another less recommended option is described [here](https://stackoverflow.com/a/66280983/2028877).


## Convert a channel into an async stream

You need to add tokio_stream to your Cargo.toml file.
```shell
cargo add tokio_stream
```

You can transform your channel with a ReceiverStream as follow.
```rust
let stream = tokio_stream::wrappers::ReceiverStream::new(receiver);
```

Then you can (for example) create batches from this async stream.
```rust
stream
  .chunks_timeout(batch_size, batch_duration)
  .for_each(|batch| {
    // do something with the batch
    This closure must return a future
  }).await;
```


## Async stream with return value on termination

Sometimes we need to communicate a return value when leaving an async stream.
A oneshot tokio channel can be used for this purpose.
```rust
TBD oneshot
```

## Create a transactional receiver for a MPSC tokio channel

List of crates to declare in you Cargo.toml.
```shell
cargo add tokio futures tokio_stream
```

The code below defines a transactional receiver for a bounded MPSC tokio channel.

```rust
use tokio::sync::mpsc::channel;
use tokio::sync::mpsc::{Sender,Receiver};
use futures::StreamExt;
use tokio_stream::wrappers::ReceiverStream;
use futures::prelude::stream::{Peekable, Peek};
use core::pin::Pin;

pub struct TransactionalChannel<T> {
    sender: Sender<T>,
    receiver: Peekable<ReceiverStream<T>>,
}

impl<T> TransactionalChannel<T> {
    pub fn new(max_capacity: usize) -> (Sender<T>, TransactionalReceiver<T>) {
        let (sender, receiver) = channel(max_capacity);

        (sender, TransactionalReceiver::new(receiver))
    }
}

pub struct TransactionalReceiver<T> {
    receiver: Peekable<ReceiverStream<T>>
}

impl<T> TransactionalReceiver<T> {
    fn new(receiver: Receiver<T>) -> Self {
        Self { receiver: tokio_stream::wrappers::ReceiverStream::new(receiver).peekable() }
    }

    pub async fn recv(mut self: Pin<&mut Self>) -> Option<&T> {
        Pin::new(&mut Pin::into_inner(self).receiver).peek().await
    }

    pub async fn commit(&mut self) -> Option<T> {
        self.receiver.next().await
    }
}
```

Now to use it.
```rust
use core::pin::Pin;

// ...
let (sender, mut receiver) = TransactionalChannel::new(100);

loop {
  let value = Pin::new(&mut receiver).recv().await;
  // do some processing with the value
  let success = do_some_processing(value);
  
  if success {
    // if this processing is successful then 
    receiver.commit().await;
  } else {
    // if this processing failed or need to be reconfigured 
    reconfigure_processing(value);
    // no commit so the next call to recv will be the same unprocessed data
  }  
}
```

## Connect a MPSC tokio channel with a fallible stream consumer


A gRPC endpoint taking a stream as parameter is an example of fallible stream consumer. In many situations (network partition, timeout, server errors, ...) you may need to retry the call to the gRPC endpoint with the same channel. Unfortunately the receiver side of your channel is consumed once transformed into the stream on the first call. The following code is an example of solution to reuse the same receiver for each retry. For simplicity, the `channel_processor` method below represents a gRPC endpoint.

```rust
use std::time::Duration;
use tokio::join;
use futures::{StreamExt, Stream};
use std::pin::Pin;
use tokio::sync::mpsc::Receiver;
use tokio::task::JoinHandle;
use std::task::{Context, Poll};
use tokio::sync::oneshot::error::TryRecvError;

#[tokio::main]
async fn main() {
    let (sender, receiver) = tokio::sync::mpsc::channel(1000);

    let task = channel_processor(receiver).await;

    let mut counter = 0;
    loop {
        sender.send(format!("ack_id_{}", counter)).await;
        counter += 1;
        tokio::time::sleep(Duration::from_millis(100)).await;
    }

    join!(task);
}

async fn channel_processor(receiver: Receiver<String>) -> JoinHandle<()> {
    tokio::spawn(async move {
        let mut reusable_receiver = ReusableReceiver::new(receiver);

        while let Err(error) = fallible_stream_consumer(reusable_receiver.stream()).await {
            // Do something with the error, i.e. logging
        }
    })
}

async fn fallible_stream_consumer(mut stream: Pin<Box<dyn Stream<Item=String> + Send + Sync + 'static>>) -> Result<(), &str> {
    let mut countdown_before_failure = 10;

    loop {
        if let Some(value) = stream.next().await {
            println!("process {:?}", value);

            countdown_before_failure -= 1;
            if countdown_before_failure == 0 {
                return Err("SomeError");
            }
        } else {
            return Ok(());
        }
    }
}


pub struct ReusableReceiver<T> {
    receiver: Option<Receiver<T>>,
    receiver_oneshot: Option<tokio::sync::oneshot::Receiver<Receiver<T>>>,
}

pub struct ReusableReceiverStream<T> {
    receiver: Option<Receiver<T>>,
    sender_oneshot: Option<tokio::sync::oneshot::Sender<Receiver<T>>>,
}

impl<T> ReusableReceiver<T> where T: Sync + Send + 'static {
    pub fn new(receiver: Receiver<T>) -> Self {
        Self {
            receiver: Some(receiver),
            receiver_oneshot: None,
        }
    }

    pub fn stream(&mut self) -> Pin<Box<dyn Stream<Item=T> + Send + Sync + 'static>> {
        if self.receiver.is_none() {
            self.recover_receiver();
        }

        let (sender_oneshot, receiver_oneshot) = tokio::sync::oneshot::channel();

        self.receiver_oneshot = Some(receiver_oneshot);

        Box::pin(ReusableReceiverStream {
            receiver: std::mem::take(&mut self.receiver),
            sender_oneshot: Some(sender_oneshot),
        })
    }

    fn recover_receiver(&mut self) {
        let mut receiver_oneshot = std::mem::take(&mut self.receiver_oneshot)
            .expect("unexpected situation, need to be fixed");

        loop {
            match receiver_oneshot.try_recv() {
                Err(TryRecvError::Closed) => {
                    return;
                }
                Err(TryRecvError::Empty) => {}
                Ok(receiver) => {
                    self.receiver = Some(receiver);
                    return;
                }
            }
        }
    }
}

impl<T> Drop for ReusableReceiverStream<T> {
    fn drop(&mut self) {
        let receiver = std::mem::take(&mut self.receiver)
            .expect("unexpected situation, need to be fixed");
        let sender_oneshot = std::mem::take(&mut self.sender_oneshot);
        let result = sender_oneshot
            .expect("unexpected situation, need to be fixed")
            .send(receiver);
        if let Err(error) = result {
            println!("{:?}", error);
        }
    }
}

impl<T> Stream for ReusableReceiverStream<T> {
    type Item = T;

    fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        self.receiver
            .as_mut()
            .expect("unexpected situation, need to be fixed")
            .poll_recv(cx)
    }
}
```
