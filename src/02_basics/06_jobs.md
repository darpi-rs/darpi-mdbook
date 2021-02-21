# Jobs

`Jobs` represent some work packaged up in a function that the application
has to execute but it's not necessary to hold up the response to the user.
Just like `middleware`, `jobs` are bound to a `Request` or a `Response` and
utilize the same argument system as `middleware`.

We have 3 types of `jobs` based on their behaviour.
This is necessary to have an optimal way of using our resources.

- Job
  - `Future` that does not block and does not perform any significant intensive work
  . It is queued on the regular tokio runtime.
  - `CpuBound` is a type of function that would do heavy computation and does not benefit from the tokio runtime.
  We would execute those on the `Rayon` runtime that is optimized for that.
  - `IOBlocking` is suitable for various kinds of IO operations that cannot be performed asynchronously.
  It is executed on a separate tokio blocking threads.


While i recommend using the macros. If you really want to, you could implement the Job factory request and response traits.

```rust 
pub enum Job {
    Future(Pin<Box<dyn Future<Output = ()> + Send>>),
    CpuBound(Box<dyn Fn() + Send>),
    IOBlocking(Box<dyn Fn() + Send>),
}
```

Few short examples.
    
```rust
use darpi::{job_factory, job::Job};

#[job_factory(Request)]
async fn first_async_job() -> Job {
  Job::Future(async { println!("first job in the background.") }.boxed())
}
```

// blocking here is ok!
```rust
use darpi::{job_factory, job::Job, Response, Body};

#[job_factory(Response)]
async fn first_sync_job(#[response] r: &Response<Body>) -> Job {
  let status_code = r.status();
  Job::IOBlocking(Box::new(move || {
    std::thread::sleep(std::time::Duration::from_secs(2));
    println!(
      "first_sync_job in the background for a request with status {}",
      status_code
    );
  }))
}
```

```rust
use darpi::{job_factory, job::Job};

#[job_factory(Response)]
async fn first_sync_job1() -> Job {
  Job::CpuBound(Box::new(|| {
    let mut r = 0;
    for _ in 0..10000000 {
      r += 1;
    }
    println!("first_sync_job1 finished in the background. {}", r)
  }))
}
```