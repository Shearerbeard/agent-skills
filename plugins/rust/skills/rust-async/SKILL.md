---
name: rust-async
description: |
  Use when writing, reviewing, or designing async and explicitly
  multithreaded Rust code — "async function", "tokio", "spawn a
  task", "Send+Sync", "Send + Sync", "multithreaded", "blocking
  the executor", "tokio::spawn", "spawn_blocking", "async/await",
  "async Rust". Contains cooperative scheduling rules, Send +
  'static bounds, spawn_blocking vs rayon vs dedicated threads,
  std::sync::Mutex vs tokio::sync::Mutex, async Drop patterns,
  and channel/stream guidance. Pair with rust-design for type
  modeling and rust-quality for anti-pattern prevention.
compatibility: claude-code opencode
license: MIT
---

# Rust Async — Concurrency Patterns

Tokio uses cooperative scheduling. Tasks yield at `.await`. A task that
spends too long without `.await` starves every other task on the same
thread.

## Cooperative Scheduling

**Don't block the executor.** Tokio's thread pool is small (1 thread per
CPU core). Blocking one thread means other tasks on that thread freeze.
Rule of thumb: no more than 10–100µs between `.await` points.

```rust
// BAD: blocks the entire thread pool
std::thread::sleep(Duration::from_secs(1));
// GOOD: yields control back to the runtime
tokio::time::sleep(Duration::from_secs(1)).await;
```

Never call blocking I/O (`std::fs`, synchronous database drivers) from
async tasks without `spawn_blocking`. See: [Alice Ryhl: What Is
Blocking?](https://ryhl.io/blog/async-what-is-blocking/).

## Send + 'static Bounds

`tokio::spawn` requires `Future + Send + 'static`. Any data held **across**
an `.await` point inside the spawned future must be `Send`. The compiler
is conservative — scoping non-`Send` values in a block that ends before
`.await` fixes most errors.

```rust
// DOES NOT COMPILE: Rc is !Send, held across .await
tokio::spawn(async {
    let rc = Rc::new("data");
    some_async_fn().await;  // rc still alive here → future is !Send
});

// FIX: drop before .await via block scope
tokio::spawn(async {
    {
        let rc = Rc::new("data");
        // use rc here
    } // rc dropped
    some_async_fn().await;  // ok
});

// PREFERRED: Arc over Rc, std::sync::Mutex over RefCell
let data = Arc::new(Mutex::new("data"));
tokio::spawn(async move {
    let guard = data.lock().unwrap();
    // DON'T hold guard across .await — scope it
    drop(guard);
    some_async_fn().await;
});
```

**`'static` bound**: spawned tasks can't borrow from the caller. Use
`Arc` for shared ownership, `move` closures to take ownership, or
scoped tasks (`tokio::task::LocalSet` + `tokio::task::spawn_local` for
single-threaded runtimes). See: [Tokio spawning
tutorial](https://tokio.rs/tokio/tutorial/spawning), [Rust async book
§Send Approximation](https://rust-lang.github.io/async-book/07_workarounds/03_send_approximation.html).

## spawn_blocking vs rayon vs Dedicated Threads

From [Alice Ryhl's cheat sheet](https://ryhl.io/blog/async-what-is-blocking/):

| Workload | Use |
|----------|-----|
| Sync I/O (file, db drivers) | `tokio::task::spawn_blocking` |
| CPU-bound computation (batch) | `rayon` — thread pool sized to CPU cores |
| CPU-bound computation (few) | `spawn_blocking` is fine, simple |
| Long-lived listener loop | Dedicated `std::thread::spawn` |

```rust
// Sync file I/O offloaded to blocking thread pool
let data = tokio::task::spawn_blocking(|| std::fs::read_to_string("large.json"))
    .await
    .unwrap()?;

// CPU-bound work via rayon
let (tx, rx) = tokio::sync::oneshot::channel();
rayon::spawn(move || { let _ = tx.send(expensive_compute()); });
let result = rx.await.unwrap();
```

## std::sync::Mutex vs tokio::sync::Mutex

**Prefer `std::sync::Mutex`** for all cases except when the lock must be
held across `.await`. `tokio::sync::Mutex` is designed for that specific
case — it yields rather than blocks. It's slower and harder to reason
about. Most async code should lock, operate, and drop the guard before
`.await`.

```rust
// GOOD: std mutex, guard dropped before .await
{
    let mut data = shared.lock().unwrap();
    *data = compute_new_value(&data);
} // guard dropped
async_work().await;

// ONLY when guard must span .await:
let mut data = shared.lock().await;
*data = fetch_update(&data).await?;  // lock held across network call
drop(data);
```

## Async Drop

`Drop` is synchronous — you can't `.await` inside it. If cleanup needs
async work, use a **channel** to hand off to a background task:

```rust
struct Checkout<C> {
    conn: Option<C>,
    return_tx: mpsc::UnboundedSender<C>,
}

impl<C> Drop for Checkout<C> {
    fn drop(&mut self) {
        if let Some(conn) = self.conn.take() {
            let _ = self.return_tx.send(conn); // non-blocking send
        }
    }
}
// Background task does the async cleanup:
while let Some(conn) = return_rx.recv().await {
    conn.graceful_shutdown().await;
}
```

## Tool Selection

| Need | Use |
|------|-----|
| Spawn N tasks, collect in order | `JoinSet` — `spawn` + `join_next` loop |
| Run N futures, collect all | `futures::future::join_all` |
| Run N futures, stop on first error | `futures::future::try_join_all` |
| Race multiple futures | `tokio::select!` |
| Stream processing | `FuturesUnordered` + `StreamExt` |
| Fixed number, different types | `tokio::join!` / `tokio::try_join!` |

```rust
// Dynamic spawn + collect as they finish
let mut set = JoinSet::new();
for url in urls {
    set.spawn(async move { reqwest::get(url).await?.text().await });
}
while let Some(result) = set.join_next().await {
    results.push(result??);
}
```

## References

- [Tokio spawning tutorial](https://tokio.rs/tokio/tutorial/spawning)
- [Alice Ryhl: What Is Blocking?](https://ryhl.io/blog/async-what-is-blocking/)
- [Rust async book §Send Approximation](https://rust-lang.github.io/async-book/07_workarounds/03_send_approximation.html)
- [Tokio shared state tutorial](https://tokio.rs/tokio/tutorial/shared-state)
