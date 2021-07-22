---
title: "Awful At Rust: Using AWS with Rusoto"
date: 2021-07-21T22:01:15-06:00
draft: true
---

### Hi, I'm Chris and I'm awful at Rust.

I recently started learning and using Rust.  For my first real project, I wanted to build a tool for sending and receiving some AWS SQS messages.  Amazon recently announced support for Rust through an [official SDK](https://github.com/awslabs/aws-sdk-rust), but at the time of this writing it is in early alpha stages and only supports a select set of services, none of which I'm familiar. I'm excited to see the support from Amazon and look forward to using it in the future, but I want to use AWS now.  Luckily, a community-driven project called [Rusoto](https://rusoto.org/index.html) provides an SDK for many services on AWS.  Because I'm new and awful at Rust, this post is my bumbling through making a call to AWS to get the SQS queue URL when given the name of the queue as an argument.

First, I need some dependencies.

```toml
# (Cargo.toml)
[dependencies]
rusoto_core = "0.47.0"
rusoto_sqs = "0.47.0"
structopt = "0.3.22"
```

From this, you can glean that Rusoto is separated roughly into a crate-per-service structure.  You can get a list of supported services and crate names at [https://rusoto.org/supported-aws-services.html](https://rusoto.org/supported-aws-services.html).

Something I haven't figured out yet is referencing structs and functions in indirect dependencies, so even though `rusoto_sqs` relies on `rusoto_core` and is even downloaded and compiled, I can't access the "Region" enum from the core without explicitly adding it as a dependency to my project.  Again, I'm awful at Rust.

Also, I'm using [StructOpt](https://docs.rs/structopt/0.3.22/structopt/index.html) to parse command line arguments for me.

So, my initial attempt at code looks like this:

```rust
use structopt::StructOpt;
use rusoto_sqs::{GetQueueUrlRequest, SqsClient, Sqs, GetQueueUrlError, GetQueueUrlResult};
use rusoto_core::region::Region;
use rusoto_core::RusotoError;

#[derive(Debug, StructOpt)]
#[structopt()]
struct Opt {
    #[structopt()]
    queue_name: String
}

fn main() {
    let opt: Opt = Opt::from_args();
    println!("Hello, finding SQS URL for queue name: {}", opt.queue_name);

    let request = GetQueueUrlRequest{
        queue_name: opt.queue_name.clone(),
        ..Default::default()
    };
    let sqs_client = SqsClient::new(Region::UsWest2);
    let result = sqs_client.get_queue_url(request).await;
    match result {
        Ok(queue_url_result) => {
            println!("Got a result from the service, queue url is: {}", queue_url_result.queue_url.unwrap());
        }
        Err(err) => {
            eprintln!("Error calling the service: {}", err);
        }
    }
}
```

However, the compiler helpfully points out that in order to use the `await` keyword, the containing function must be an `async` function.

```
13 | fn main() {
   |    ---- this is not `async`
...
22 |     let result = sqs_client.get_queue_url(request).await;
   |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ only allowed inside `async` functions and blocks
```

Changing the main function to async is pretty simple:

```rust
async fn main() {
```

But then results in this error.

```
error[E0752]: `main` function is not allowed to be `async`
  --> src\main.rs:13:1
   |
13 | async fn main() {
   | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

Somewhere in the Rusoto documentation, I read that Rusoto uses Tokio under the hood as an async runtime.  In taking a look at Tokio, I found that by adding the `tokio::main` attribute to the main function, it converts the function to a tokio runtime function and allows the `async` keyword to be used.

So, back to `Cargo.toml`:
```toml
# (Cargo.toml)
[dependencies]
rusoto_core = "0.47.0"
rusoto_sqs = "0.47.0"
structopt = "0.3.22"
tokio = { version = "1", features = ["full"]}
```

then, add the attribute to `main`:

```rust
#[tokio::main]
async fn main() {
```

