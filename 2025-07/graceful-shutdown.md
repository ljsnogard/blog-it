# 在 Rust 中如何优雅地取消异步任务

本文也在 [github](https://github.com/ljsnogard/blog-it/tree/main/2025-07/graceful-shutdown.md) 上发布，敬请关注勘误或讨论。

## 什么是优雅

在 Rust 中要取消一个异步操作，如果这个异步操作已经被 `Future` 所包装，那么只需要 `drop()`。依托于 RAII 语义，和异步任务相关的资源通常都会被恰当地释放。如果是在其他线程运行的任务，通过 detach 或者同样简单地 drop 线程，也可以达到类似的效果。  

这些都可以归类为不太优雅的取消方式。当然，对于不关心资源管理的任务调用者来说可能反而是优雅的，因为业务代码中不需要关心丰富而复杂的细节。但是对于异步任务实现者，或者库作者来说，评价的立场则是完全相反的。  

关于什么是优雅，在 tokio 中式具有明确的实践——[Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown)。

在本文中，我们采取类似的理念来定义，什么是优雅地取消异步任务：
1. 让异步任务接收取消的信号
2. 让调用者决定何时发送取消的信号
3. 让调用者等待异步任务取消

## 为何要优雅

在 tokio 的文档或者事件中，我们可以看到优雅取消任务针对的都是比较重量级的应用场景，譬如说，优雅地关闭一个服务器。  
在这种涉及多种资源甚至涉及到服务器集群维护的场景中，显然是有价值的，例如，可以更快地关闭 TCP 连接，更快地让操作系统回收资源，更快地让服务器集群发现节点失效，等等。  
因为 Rust 的 RAII 语义，我们通常不需要担心资源的最终回收，除非这些资源很难依靠**单机上的 RAII 语义**来保证。  
但是更小尺度的应用中，这样的优雅是否也有实践价值呢？  

考虑如下 trait 所表示的简单的读取数据的需求:

```rust
pub trait Input<T = u8> {
    type Err;

    fn read_async<'f>(
        &'f mut,
        buf: &'f mut [MaybeUninit<T>],
    ) -> impl Future<Output = Result<usize, Self::Err>>;
}
```

现在你需要为一个非常慢速且不稳定的网络设备，实现 `Input` 接口，使用者有可能在过长的异步等待过程中，取消等待**但继续保留任何已接收的数据**。  
问题来了，如果通过不优雅的方式取消 `read_async`，例如直接 drop `Future` 实例，那么调用者也将无法得知已接收的数据到底有多少，因为调用者无法得到 Future 所承诺的 Result 正常结果，即 buf 被填充的长度。  

```rust
let f = input.read_async(&mut buf);
drop(f); // 无法获得 buf 被写入的数量
```

对问题作更进一步的抽象，在取消任务之前能获得任务完成的进度，本质上就是支持轮询的异步任务——在任何进度的时候被打断或完成前，获得一个任务进度。  
如果你有关注异步 IO 的新进展，你可能听说过 `io_uring`，它要解决的问题和这类需求有相似之处。  

## 如何才优雅

你可能马上指出，是 `Input` 的设计有问题，导致了它实质上不能支持这类支持轮询的异步需求。要正确地描述这类需求，`Input` 应该这样设计：

```rust
/// 新增一个对 CancelToken 的定义
pub trait CancelToken;

pub trait Input<T = u8> {
    type Err;

    fn read_async<'f>(
        &'f mut,
        buf: &'f mut [MaybeUninit<T>],
        tok: &'f mut impl CancelToken,
    ) -> impl Future<Output = Result<usize, Self::Err>>;
}
```

这个设计确实更反映了可取消的异步任务的本质，它正确地描述了新的依赖，即对 `CancelToken` 的依赖。通过 `CancelToken` ，我们，或者说 `read_async` 的实现者，可以优雅地完成任务，即：

1. 让异步任务接收取消的信号
2. 让调用者决定何时发送取消的信号
3. 让调用者等待异步任务取消

但是考虑到 Rust 语言并不支持默认参数，具体来说，不支持像在 C# 中的这种定义和写法：

```Csharp
public interface IInput {
    async Task<int> ReadAsync(
        Memory<byte> buf, 
        CancellationToken tok = default); // 使用了默认参数
}

if (aDevice is IInput input) {
    var x = await input.ReadAsync(buf); // 默认情况不取消，不需要传递 CancellationToken 
}
```

而且将 `Input` 强制与一个 `CancelToken` 耦合看起来也不够优雅，所以我们可以考虑 Rust 语言提供的另一种语法糖 `IntoFuture`。  

因为 Rust 中的 Future 和 C# 中的 Task 或者异步任务不一样，Rust 中的 Future 必须调用 `poll` 才会开始被执行，而在绝大多数的情况下 `await` 关键字会让异步运行时替我们调用 `poll`，**还会自动调用 `IntoFuture::into_future()`**。  

通过引入一些间接层，我们可以进一步地把 `CancelToken` 隐藏到实际调用时：

```rust
pub trait TrMayCancel<'a>: IntoFuture {
    type MayCancelOutput;

    fn may_cancel_with<'f, C: TrCancellationToken>(
        self,
        token: &'f mut C,
    ) -> impl Future<Output = Self::MayCancelOutput>;
}

pub trait TrInput {
    type Err;

    fn read_async<'f>(
        &'f mut,
        buf: &'f mut [MaybeUninit<T>],
    ) -> impl TrMayCancel<'a, MayCancelOutput = Result<usize, Self::Err>>;
}
```

通过这样的设计，调用者可以选择：
```rust
let _ = input.read_async(&mut buf).await;
```
或者：
```rust
let _ = input.read_async(&mut buf).may_cancel_with(&mut tok).await;
```

## 这真的优雅吗？

我们引入了一个额外的 `IntoFuture` 和 `TrMayCancel` 层来隐藏运行时可选的 CancelToken，显然对于库设计者来说，这是增加了额外工作量的。但是看起来这些工作量可以用宏来实现。
限于篇幅，这里只直接介绍作者解决这个问题的产品 [gen_mcf_macro](https://github.com/ljsnogard/gen_mcf_macro.rs/tree/dev/0.3.0)。

总的来说，譬如库作者需要为 `MyInput` 实现 `TrInput<u8>`，那么可以通过编写：

```rust
#[gen_may_cancel_future(MyDeviceRead)]
async fn my_device_read_async<'f, C: TrCancellationToken>(
    input: &'f mut MyInput,
    buffer: &'f mut [u8],
    token: &'f mut impl C,
) -> Result<usize, MyInputError> {
    // 实现代码
}
```

就会自动生成 `MyDeviceReadAsync` 和 `MyDeviceReadFuture` 两个 struct。  
其中`MyDeviceReadAsync`实现了 `TrMayCancel`，而 `MyDeviceReadFuture` 实现了 `Future`。
