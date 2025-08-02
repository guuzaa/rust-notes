# Arc  简介

## 介绍

`Arc` 是 `Atomically Reference Counted` 的缩写，即原子引用计数，一种线程安全的共享指针。

通过引用计数来实现共享所有权，从而实现线程安全。

因为 Arc 就是为了多线程场景设计的， 所以往往与 Mutex/RwLock/Atomic 等类型搭配使用。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    
    // 共享所有权
    let counter_clone = Arc::clone(&counter);
    
    let handle = thread::spawn(move || {
        let mut value = counter_clone.lock().unwrap();
        *value += 1;
        println!("Thread: Counter = {}", *value);
    });
    
    handle.join().unwrap();
    
    let final_value = counter.lock().unwrap();
    println!("Final counter: {}", *final_value);
}
```

```rust
use std::sync::{Arc, atomic::{AtomicI32, Ordering}};
use std::thread;

fn main() {
    let counter = Arc::new(AtomicI32::new(0));
    
    // 共享所有权
    let counter_clone = Arc::clone(&counter);
    
    let handle = thread::spawn(move || {
        counter_clone.fetch_add(1, Ordering::SeqCst);
        println!("Thread: Counter = {}", counter_clone.load(Ordering::SeqCst));
    });
    
    handle.join().unwrap();
    
    println!("Final counter: {}", counter.load(Ordering::SeqCst));
}
```

### 为什么全局变量不需要 Arc？

全局变量（static）在 Rust 中具有 `'static` 生命周期，它们在程序运行期间都存在。本身就已经是共享的，不需要额外的共享指针。

```rust
use std::sync::Mutex;

// 全局变量不需要 Arc
static COUNTER: Mutex<i32> = Mutex::new(0);

fn main() {
    let mut value = COUNTER.lock().unwrap();
    *value += 1;
    println!("Counter: {}", *value);
}
```

### 循环引用

常见于链表、树中。
循环引用会导致内存泄漏，因为引用计数永远不会为 0。
解决方法：使用 `Weak` 引用，避免循环引用。

比如下面这个例子：

```rust
struct Parent {
    name: String,
    children: Vec<Arc<Child>>,
}

struct Child {
    name: String,
    parent: Weak<Parent>,
}
```

### 最佳实践

1. **避免循环引用**: 可能导致内存泄漏
2. **合理使用**: 多线程共享时使用 Arc
3. **组合使用**: 与 Mutex、RwLock 或 Atomic 类型组合使用

### 总结

Arc 是 Rust 中实现线程安全共享所有权的核心工具。它通过原子引用计数机制，允许多个线程安全地共享同一份数据。通常与同步原语（如 Mutex、RwLock）结合使用。
