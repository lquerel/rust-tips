# Async Patterns

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

