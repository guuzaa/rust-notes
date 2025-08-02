# 在 Rust 中安全地使用全局变量

## 介绍

C 对全局变量的使用几乎没有限制，比如：

```c
int global_var = 1;

void set_global_var(int value) {
    global_var = value;
}

```

```c
int global_nums[2] = {1, 2};

void set_global_nums(int index, int value) {
    global_nums[index] = value;
}

int get_global_nums(int index) {
    return global_nums[index];
}
```

在多线程场景下，全局变量可能会导致数据竞争，从而引发未定义行为。

为此，Rust 中全局变量默认是不可变的，这意味着一旦被初始化，就不能再被修改。

如果要修改全局变量，一种简单粗暴的方法就是，使用 `unsafe` 块，比如：

```rust
static mut GLOBAL_VAR: i32 = 1;

fn set_global_var(value: i32) {
    unsafe {
        GLOBAL_VAR = value;
    }
}

fn main() {
    set_global_var(2);
    println!("GLOBAL_VAR: {}", unsafe { GLOBAL_VAR });
}
```

```rust
static mut GLOBAL_NUMS: [i32; 2] = [1, 2];

fn set_global_nums(index: usize, value: i32) {
    unsafe {
        GLOBAL_NUMS[index] = value;
    }
}

fn get_global_nums(index: usize) -> i32 {
    unsafe {
        GLOBAL_NUMS[index]
    }
}

fn main() {
    set_global_nums(0, 2);
    set_global_nums(1, 3);
    println!("GLOBAL_NUMS[0]: {}", get_global_nums(0));
    println!("GLOBAL_NUMS[1]: {}", get_global_nums(1));
}
```

如果你确定没有多线程场景，其实可以到此结束。但是我们其实并不能准确判断是否存在多线程场景——你不知道你不知道，所以需要更安全的做法。

下面我们将根据使用场景逐一介绍更安全的做法——把上面的 `unsafe` 块去掉。

## 方法 1：Mutex

我们可以借助 `Mutex` 来实现全局变量的安全访问。

```rust
use std::sync::Mutex;

static GLOBAL_VAR: Mutex<i32> = Mutex::new(1);

fn set_global_var(value: i32) {
    *GLOBAL_VAR.lock().unwrap() = value;
}

fn main() {
    set_global_var(2);
    println!("GLOBAL_VAR: {}", *GLOBAL_VAR.lock().unwrap());
}
```

```rust
use std::sync::Mutex;

static GLOBAL_NUMS: Mutex<[i32; 2]> = Mutex::new([1, 2]);

fn set_global_nums(index: usize, value: i32) {
    let mut nums = GLOBAL_NUMS.lock().unwrap();
    nums[index] = value;
}

fn get_global_nums(index: usize) -> i32 {
    let nums = GLOBAL_NUMS.lock().unwrap();
    nums[index]
}

fn main() {
    set_global_nums(0, 2);
    set_global_nums(1, 3);
    println!("GLOBAL_NUMS[0]: {}", get_global_nums(0));
    println!("GLOBAL_NUMS[1]: {}", get_global_nums(1));
}
```
>  这与多线程才需要 Mutex 的直觉相悖

## 方法 2：OnceLock

如果全局变量只需要写一次，然后读取多次，那可以不用 `Mutex` —— 每次在读时都要锁操作。

我们可以使用 `OnceLock` ，还可以避免误修改。

```rust
use std::sync::OnceLock;

static GLOBAL_VAR: OnceLock<i32> = OnceLock::new();

fn set_global_var(value: i32) {
    GLOBAL_VAR.get_or_init(|| {
        value
    });
}

fn main() {
    set_global_var(2);
    println!("GLOBAL_VAR: {:?}", GLOBAL_VAR.get());
    set_global_var(3);
    println!("GLOBAL_VAR: {:?}", GLOBAL_VAR.get());
}
```

## 方法 3：RwLock

如果全局变量写多次，但是读取次数远少于写入次数，我们可以使用 `RwLock` 来实现。

```rust
use std::sync::RwLock;

static GLOBAL_VAR: RwLock<i32> = RwLock::new(1);

fn set_global_var(value: i32) {
    *GLOBAL_VAR.write().unwrap() = value;
}

fn main() {
    set_global_var(2);
    println!("GLOBAL_VAR: {:?}", *GLOBAL_VAR.read().unwrap());
}
```

```rust
use std::sync::RwLock;

static GLOBAL_NUMS: RwLock<[i32; 2]> = RwLock::new([1, 2]);

fn set_global_nums(index: usize, value: i32) {
    let mut nums = GLOBAL_NUMS.write().unwrap();
    nums[index] = value;
}

fn get_global_nums(index: usize) -> i32 {
    let nums = GLOBAL_NUMS.read().unwrap();
    nums[index]
}

fn main() {
    set_global_nums(0, 2);
    set_global_nums(1, 3);
    println!("GLOBAL_NUMS[0]: {}", get_global_nums(0));
    println!("GLOBAL_NUMS[1]: {}", get_global_nums(1));
}
```

## 方法 4：Atomic

如果全局变量是整型，那我们可以使用 `Atomic` 来实现。

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static GLOBAL_VAR: AtomicI32 = AtomicI32::new(1);

fn set_global_var(value: i32) {
    GLOBAL_VAR.store(value, Ordering::Relaxed);
}

fn main() {
    set_global_var(20);
    println!("GLOBAL_VAR: {}", GLOBAL_VAR.load(Ordering::Relaxed));

    let h1 = std::thread::spawn(move || {
        GLOBAL_VAR.store(30, Ordering::Relaxed);
    });
    let h2 = std::thread::spawn(move || {
        GLOBAL_VAR.store(40, Ordering::Relaxed);
    });

    h1.join().unwrap();
    h2.join().unwrap();

    println!("GLOBAL_VAR: {}", GLOBAL_VAR.load(Ordering::Relaxed));
}
```

## 补充：动态类型

如果全局变量是需要使用动态内存的类型，那么就不能直接使用 Mutex/RwLock/OnceLock 了。
因为动态类型的初始化操作不是 const，非 const 函数不允许在 static 中使用。

我们可以使用 `LazyLock` 来实现，比如:

```rust
use std::{collections::HashMap, sync::{LazyLock, RwLock}};

static GLOBAL_VAR: LazyLock<RwLock<HashMap<String, i32>>> = LazyLock::new(|| RwLock::new(HashMap::new()));

fn set_global_var(key: String, value: i32) {
    let mut handler = GLOBAL_VAR.write().unwrap();
    handler.insert(key, value);
}

fn main() {
    set_global_var("key1".to_string(), 10);
    println!("GLOBAL_VAR: {:?}", GLOBAL_VAR.read().unwrap());
}
```

## 总结

|  方法 | 特点 | 缺点 |
| -------- | ------------ | ---------------- |
| `Mutex` | 每次读写都要锁操作 | 需要处理**加锁失败**场景 |
| `OnceLock` | 只能写一次，读取多次 |  场景受限，只能写一次 |
| `RwLock` | 读写锁， 可以同时读取 | 需要处理**加锁失败**场景 |
| `Atomic` | 原子操作，性能较好 | 只能修改整型，**内存序需要仔细** |
| `unsafe` | 简单，性能好 | 需要处理**未定义行为** |


## 参考

[static 关键字](https://doc.rust-lang.org/std/keyword.static.html)
