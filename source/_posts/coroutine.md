---
title: 从0到1实现基于协程的WebServer(1) —— coroutine协程
tags:
  - C++
  - 协程
categories:
  - WebServer
abbrlink: 77a02364
date: 2024-06-19 14:35:50
---
# 什么是协程

> 协程是能暂停执行以在之后恢复的函数。

协程分为**有栈协程**和**无栈协程**。有栈协程在协程之间存在函数调用栈，保存了单独的上下文环境，可在嵌套函数中暂停执行。而无栈协程所有协程共享一个执行栈，本质上是一个状态机。

# C++20中的协程

> C++20 的协程是无栈协程，它们通过返回到调用方暂停执行，并且恢复执行所需的数据与栈分离存储。
> 协程函数中需包含 **co\_await**、**co\_yield**、**co\_return** 三个表达式的中的至少一个。

## 使用 co_return 的协程
co_return 用于结束协程，并可返回一个值。
### 无返回值的协程
```cpp
#include <coroutine>
#include <iostream>

struct Promise {
  auto initial_suspend() noexcept {
    std::cout << "initial_suspend" << std::endl;
    return std::suspend_always{};
  }
  auto final_suspend() noexcept {
    std::cout << "final_suspend" << std::endl;

    return std::suspend_always{};
  }
  void unhandled_exception() {}
  auto get_return_object() {
    return std::coroutine_handle<Promise>::from_promise(*this);
  }

  void return_void() { std::cout << "Hello Coroutine!" << std::endl; }
};

struct Task : std::coroutine_handle<Promise> {
  using promise_type = Promise;
  Task(std::coroutine_handle<promise_type> handle)
      : std::coroutine_handle<Promise>(handle) {}
};

Task hello() {
  std::cout << "hello()" << std::endl;
  co_return;
}

int main() {
  auto t = hello();
  std::cout << "ready to resume" << std::endl;
  t.resume();
}
```
以上代码的执行结果为
```bash
initial_suspend
ready to resume
hello()
Hello Coroutine!
final_suspend
```
一个协程类型必须包含一个 **promise** 对象：
1. 协程通过 **promise** 对象提交其结果或异常。在协程开始执行时会先调用 `promise.initial_suspend()` 并 `co_await` 其结果。(`std::suspend_always` 是一个 `awaiter` 表示`co_await` 总是暂停并且不产生值)。
2. 协程函数在运行到 `co_return` 时退出协程，并且在无返回值时会执行 `promise.return_void()`
3. 协程函数结束时调用 `promise.final_suspend()` 并 `co_await` 它的结果。
### 具有返回值的协程
```cpp
#include <coroutine>
#include <iostream>

struct Promise {
  auto initial_suspend() noexcept {
    std::cout << "initial_suspend" << std::endl;
    return std::suspend_always{};
  }
  auto final_suspend() noexcept {
    std::cout << "final_suspend" << std::endl;

    return std::suspend_always{};
  }
  void unhandled_exception() {}
  auto get_return_object() {
    return std::coroutine_handle<Promise>::from_promise(*this);
  }

  void return_value(int val) {
    m_val = val;
    std::cout << "return_value: " << val << std::endl;
  }

  int m_val;
};

struct Task : std::coroutine_handle<Promise> {
  using promise_type = Promise;
  Task(std::coroutine_handle<promise_type> handle)
      : std::coroutine_handle<Promise>(handle) {}
};

Task hello() {
  std::cout << "hello()" << std::endl;
  co_return 42;
}

int main() {
  auto t = hello();
  std::cout << "ready to resume: " << t.promise().m_val << std::endl;
  t.resume();
  std::cout << "resume done: " << t.promise().m_val << std::endl;
}
```
执行结果如下：
```bash
initial_suspend
ready to resume: 0
hello()
return_value: 42
final_suspend
resume done: 42
```
对于具有非空类型参数（假设为 `T val`）的 `co_return val` 会调用 `promise.return_value(T val)`。可以通过在 `promise` 中自定义成员变量 `m_val` 来实现获取返回值。并可以在协程外部通过 `promise` 访问。
## 使用 co_await 的协程
`co_await` 暂停协程并将控制权交给调用者
```cpp
#include <coroutine>
#include <iostream>

struct PreAwaiter {
  bool await_ready() const noexcept { return false; }

  std::coroutine_handle<>
  await_suspend(std::coroutine_handle<> handle) const noexcept {
    std::cout << "PreAwaiter await_suspend" << std::endl;
    if (m_handle) {
      return m_handle;
    }
    return std::noop_coroutine();
  }

  void await_resume() const noexcept {}
  std::coroutine_handle<> m_handle;
};

struct Promise {
  auto initial_suspend() noexcept { return std::suspend_always{}; }
  auto final_suspend() noexcept { return PreAwaiter{m_pre}; }
  void unhandled_exception() {}
  auto get_return_object() {
    return std::coroutine_handle<Promise>::from_promise(*this);
  }

  void return_value(int val) {
    m_val = val;
    std::cout << "return_value: " << val << std::endl;
  }

  int m_val;
  std::coroutine_handle<> m_pre;
};

struct Awaiter {
  bool await_ready() const noexcept { return false; }

  auto await_suspend(std::coroutine_handle<> handle) const noexcept {
    std::cout << "await_suspend" << std::endl;
    m_handle.promise().m_pre = handle;
    return m_handle;
  }

  int await_resume() const noexcept { return m_handle.promise().m_val; }
  std::coroutine_handle<Promise> m_handle;
};

struct Task : std::coroutine_handle<Promise> {
  using promise_type = Promise;
  Task(std::coroutine_handle<promise_type> handle)
      : std::coroutine_handle<Promise>(handle) {}

  auto operator co_await() {
    return Awaiter{*static_cast<std::coroutine_handle<Promise> *>(this)};
  }
};

Task world() {
  std::cout << "world" << std::endl;
  co_return 1;
}

Task hello() {
  std::cout << "hello" << std::endl;
  auto i = co_await world();
  co_return i + 1;
}

int main() {
  auto t = hello();
  std::cout << "ready to resume: " << t.promise().m_val << std::endl;
  while (!t.done()) {
    t.resume();
  }
}
```
以上代码执行结果为
```bash
ready to resume: 0
hello
await_suspend
world
return_value: 1
PreAwaiter await_suspend
return_value: 2
PreAwaiter await_suspend
```
可以看到hello任务在 `co_await world()`时，会先调用 `awaiter.await_ready()`，如果为 `false` 则：
1. 暂停原协程的执行
2. 调用 `awaiter.await_suspend(handle)` 其中 `handle` 是表示原协程的协程句柄。根据其返回值不同有不同行为：
    - 返回 `void`、`true`、`noop_coroutine_handle`：将控制返回给原协程的调用者，保持原协程暂停
    - 返回 `fasle`；恢复原协程的执行
    - 返回某个协程句柄：通过调用 `handle.resume()` 恢复该句柄、
3. 最后调用 `awaiter.await_resume()` 将其的结果作为 `co_await` 表达式的返回值

在本例中 `co_await world()` 实际上返回的是 `world()` 这个 `Task` 任务的协程句柄，这是由于 `Awaiter` 在初始化时将 `m_handle` 设置为 `Task` 的协程句柄。故此时 `co_await world()` 时会进行 `world()` 协程的执行。
```cpp
struct Awaiter {
  bool await_ready() const noexcept { return false; }

  auto await_suspend(std::coroutine_handle<> handle) const noexcept {
    std::cout << "await_suspend" << std::endl;
    m_handle.promise().m_pre = handle;
    return m_handle;
  }

  int await_resume() const noexcept { return m_handle.promise().m_val; }
  std::coroutine_handle<Promise> m_handle;
};

struct Task : std::coroutine_handle<Promise> {
  using promise_type = Promise;
  Task(std::coroutine_handle<promise_type> handle)
      : std::coroutine_handle<Promise>(handle) {}

  auto operator co_await() {
    return Awaiter{*static_cast<std::coroutine_handle<Promise> *>(this)};
  }
};
```
并且在 `co_await awaiter` 时将 `m_handle.promise().m_pre` 设置为了当前协程的句柄，则在 `m_handle` 执行结束时会 `co_await promise.final_suspend()` 恢复`m_pre` 协程句柄。
```cpp
struct PreAwaiter {
  bool await_ready() const noexcept { return false; }

  std::coroutine_handle<>
  await_suspend(std::coroutine_handle<> handle) const noexcept {
    std::cout << "PreAwaiter await_suspend" << std::endl;
    if (m_handle) {
      return m_handle;
    }
    return std::noop_coroutine();
  }

  void await_resume() const noexcept {}
  std::coroutine_handle<> m_handle;
};

struct Promise {
  auto initial_suspend() noexcept { return std::suspend_always{}; }
  auto final_suspend() noexcept { return PreAwaiter{m_pre}; }
  void unhandled_exception() {}
  auto get_return_object() {
    return std::coroutine_handle<Promise>::from_promise(*this);
  }

  void return_value(int val) {
    m_val = val;
    std::cout << "return_value: " << val << std::endl;
  }

  int m_val;
  std::coroutine_handle<> m_pre;
};
```
## 使用 co_yield 的协程
co_yield 表达式向调用方返回一个值并暂停当前协程。通过定义 `promise.yield_value` 来定义其行为，并在最后 `co_await` 其返回值。
```cpp
#include <coroutine>
#include <iostream>

struct Promise {
  auto initial_suspend() noexcept {
    std::cout << "initial_suspend" << std::endl;
    return std::suspend_always{};
  }
  auto final_suspend() noexcept {
    std::cout << "final_suspend" << std::endl;

    return std::suspend_always{};
  }
  void unhandled_exception() {}
  auto get_return_object() {
    return std::coroutine_handle<Promise>::from_promise(*this);
  }

  auto yield_value(int val) {
    std::cout << "yield: " << val << std::endl;
    return std::suspend_always{};
  }

  void return_void() {}
};

struct Task : std::coroutine_handle<Promise> {
  using promise_type = Promise;
  Task(std::coroutine_handle<promise_type> handle)
      : std::coroutine_handle<Promise>(handle) {}
};

Task hello() {
  std::cout << "hello()" << std::endl;
  for (int i = 0; i < 10; i++) {
    co_yield i;
  }
}

int main() {
  auto t = hello();
  while (!t.done()) {
    t.resume();
  }
}
```
以上代码的执行结果为
```bash
initial_suspend
hello()
yield: 0
yield: 1
yield: 2
yield: 3
yield: 4
yield: 5
yield: 6
yield: 7
yield: 8
yield: 9
final_suspend
```
# 引用
- [【C++20】从0开始自制协程库，有手就行（上）](https://www.bilibili.com/video/BV1Yz421Z7rZ) 
- [archibate/co_async: C++20 Coroutine Library for Education Purpose (WIP)](https://github.com/archibate/co_async)
- [协程 (C++20) - cppreference.com](https://zh.cppreference.com/w/cpp/language/coroutines)

