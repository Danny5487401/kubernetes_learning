# Informer机制
![](.14_informer_images/informer_chain.png)
参考高清图：https://www.processon.com/view/link/5f55f3f3e401fd60bde48d31


Informer 是 client-go 中的核心工具包，已经被 kubernetes 中众多组件所使用。
所谓 Informer，其实就是一个带有本地缓存和索引机制的、可以注册 EventHandler 的 client，本地缓存被称为 Store，索引被称为 Index。
使用 informer 的目的是为了减轻 apiserver 数据交互的压力而抽象出来的一个 cache 层, 客户端对 apiserver 数据的 "读取" 和 "监听" 操作都通过本地 informer 进行

## 为什么需要 Informer 机制？
我们知道Kubernetes各个组件都是通过REST API跟API Server交互通信的，而如果每次每一个组件都直接跟API Server交互去读取/写入到后端的etcd的话，会对API Server以及etcd造成非常大的负担。
而Informer机制是为了保证各个组件之间通信的实时性、可靠性，并且减缓对API Server和etcd的负担

## Informer 的主要功能：

- 同步数据到本地缓存

- 根据对应的事件类型，触发事先注册好的 ResourceEventHandle


## Informer 需要满足哪些要求？

- 消息可靠性

- 消息实时性

- 消息顺序性

- 高性能

## Informer运行原理
![](.14_informer_images/informer_process1.png)
各个组件包括：

- Reflector：用于监控（watch）指定的资源，当监控的资源发生变化时，触发相应的变更事件。并将资源对象存放到本地缓存DeltaFIFO中
- DeltaFIFO：对资源对象的的操作类型进行队列的基本操作
  - FIFO：先进先出队列，提供资源对象的增删改查等操作
  - Dealta：资源对象存储，可以保存资源对象的操作类型。如：添加操作类型、更新操作类型、删除操作类型、同步操作类型
- Indexer：存储资源对象，并自带索引功能的本地存储。
  - Reflect从DeltaFIFO中将消费出来的资源对象存储至Indexer
  - Indexer中的数据与Etcd完全一致，client-go可以从本地读取，减轻etcd和api-server的压力

## Informer的工作流程
![](.14_informer_images/informer_process2.png)
黄色的部分是controller相关的框架，包括workqueue。蓝色部分是client-go的相关内容，包括informer, reflector(其实就是informer的封装), indexer。

1. Informer 首先会 list/watch apiserver，Informer 所使用的 Reflector 包负责与 apiserver 建立连接，Reflector 使用 ListAndWatch 的方法，
会先从 apiserver 中 list 该资源的所有实例，list 会拿到该对象最新的 resourceVersion，然后使用 watch 方法监听该 resourceVersion 之后的所有变化，
若中途出现异常，reflector 则会从断开的 resourceVersion 处重现尝试监听所有变化，一旦该对象的实例有创建、删除、更新动作，Reflector 都会收到"事件通知"，
这时，该事件及它对应的 API 对象这个组合，被称为增量（Delta），它会被放进 DeltaFIFO 中。

2. Informer 会不断地从这个 DeltaFIFO 中读取增量，每拿出一个对象，Informer 就会判断这个增量的时间类型，然后创建或更新本地的缓存，也就是 store。

3. 如果事件类型是 Added（添加对象），那么 Informer 会通过 Indexer 的库把这个增量里的 API 对象保存到本地的缓存中，并为它创建索引，若为删除操作，则在本地缓存中删除该对象。

4. DeltaFIFO 再 pop 这个事件到 controller 中，controller 会调用事先注册的 ResourceEventHandler 回调函数进行处理。

5. 在 ResourceEventHandler 回调函数中，其实只是做了一些很简单的过滤，然后将关心变更的 Object 放到 workqueue 里面。

6. Controller 从 workqueue 里面取出 Object，启动一个 worker 来执行自己的业务逻辑，业务逻辑通常是计算目前集群的状态和用户希望达到的状态有多大的区别，
然后孜孜不倦地让 apiserver 将状态演化到用户希望达到的状态，比如为 deployment 创建新的 pods，或者是扩容/缩容 deployment。

7. 在worker中就可以使用 lister 来获取 resource，而不用频繁的访问 apiserver，因为 apiserver 中 resource 的变更都会反映到本地的 cache 中。


### List & Watch
![](.14_informer_images/list_n_watch.png)

List所做的，就是向API Server发送一个http短链接请求，罗列所有目标资源的对象。而Watch所做的是实际的“监听”工作，通过http长链接的方式，其与API Server能够建立一个持久的监听关系，当目标资源发生了变化时，API Server会返回一个对应的事件，从而完成一次成功的监听，之后的事情便交给后面的handler来做。

这样一个List & Watch机制，带来了如下几个优势：
- 事件响应的实时性：通过Watch的调用，当API Server中的目标资源产生变化时，能够及时的收到事件的返回，从而保证了事件响应的实时性。而倘若是一个轮询的机制，其实时性将受限于轮询的时间间隔。

- 事件响应的可靠性：倘若仅调用Watch，则如果在某个时间点连接被断开，就可能导致事件被丢失。List的调用带来了查询资源期望状态的能力，客户端通过期望状态与实际状态的对比，可以纠正状态的不一致。二者结合保证了事件响应的可靠性。

- 高性能：倘若仅周期性地调用List，轮询地获取资源的期望状态并在与当前状态不一致时执行更新，自然也可以do the job。但是高频的轮询会大大增加API Server的负担，低频的轮询也会影响事件响应的实时性。Watch这一异步消息机制的结合，在保证了实时性的基础上也减少了API Server的负担，保证了高性能。

- 事件处理的顺序性：我们知道，每个资源对象都有一个递增的ResourceVersion，唯一地标识它当前的状态是“第几个版本”，每当这个资源内容发生变化时，对应产生的事件的ResourceVersion也会相应增加。在并发场景下，K8s可能获得同一资源的多个事件，由于K8s只关心资源的最终状态，因此只需要确保执行事件的ResourceVersion是最新的，即可确保事件处理的顺序性。


### 二级缓存
二级缓存属于 Informer 的底层缓存机制，这两级缓存分别是 DeltaFIFO 和 LocalStore。这两级缓存的用途各不相同。DeltaFIFO 用来存储 Watch API 返回的各种事件 ，LocalStore 只会被 Lister 的 List/Get 方法访问 。

如果K8s每次想查看资源对象的状态，都要经历一遍List调用，显然对 API Server 也是一个不小的负担，对此，一个容易想到的方法是使用一个cache作保存，需要获取资源状态时直接调cache，当事件来临时除了响应事件外，也对cache进行刷新。

虽然 Informer 和 Kubernetes 之间没有 resync 机制，但 Informer 内部的这两级缓存之间存在 resync 机制。


### Resync

Resync 机制会将 Indexer 的本地缓存重新同步到 DeltaFIFO 队列中。一般我们会设置一个时间周期，让 Indexer 周期性地将缓存同步到队列中。直接 list/watch API Server 就已经能拿到集群中资源对象变化的 event 了

## 源码
client-go 中提供了几种不同的 Informer：

- 通过调用 NewInformer 函数创建一个简单的不带 indexer 的 Informer。

- 通过调用 NewIndexerInformer 函数创建一个简单的带 indexer 的 Informer。

- 通过调用 NewSharedIndexInformer 函数创建一个 Shared 的 Informer。

- 通过调用 NewDynamicSharedInformerFactory 函数创建一个为 Dynamic 客户端的 Shared 的 Informer


这里的 Shared 的 Informer 引入，其实是因为随着 K8S 中，相同资源的监听者在不断地增加，
从而导致很多调用者通过 Watch API 对 API Server 建立一个长连接去监听事件的变化，这将严重增加了 API Server 的工作负载，及资源的浪费。


### SharedInformer
我们平时说的 Informer 其实就是 SharedInformer，它是可以共享使用的。如果同一个资源的 Informer 被实例化多次，那么就会运行多个 ListAndWatch 操作，这会加大 APIServer 的压力。而 SharedInformer 通过一个 map 来让同一类资源的 Informer 实现共享一个 Refelctor，这样就不会出现上面这个问题了。

Informer通过Local Store缓存目标资源对象，且仅为自己所用。但是在K8s中，一个Controller可以关心不止一种资源，使得多个Controller所关心的资源彼此会存在交集。如果几个Controller都用自己的Informer来缓存同一个目标资源，显然会导致不小的空间开销，因此K8s引入了SharedInformer来解决这个问题。

SharedInformer拥有为多个Controller提供一个共享cache的能力，从而避免资源缓存的重复、减小空间开销。除此之外，一个SharedInformer对一种资源只建立一个与API Server的Watch监听，且能够将监听得到的事件分发给下游所有感兴趣的Controller，这也显著地减少了API Server的负载压力。实际上，K8s中广泛使用的都是SharedInformer，Informer则出场甚少。

```go
type SharedInformer interface {
	// AddEventHandler adds an event handler to the shared informer using the shared informer's resync
	// period.  Events to a single handler are delivered sequentially, but there is no coordination
	// between different handlers.
	AddEventHandler(handler ResourceEventHandler)
	// AddEventHandlerWithResyncPeriod adds an event handler to the
	// shared informer with the requested resync period; zero means
	// this handler does not care about resyncs.  The resync operation
	// consists of delivering to the handler an update notification
	// for every object in the informer's local cache; it does not add
	// any interactions with the authoritative storage.  Some
	// informers do no resyncs at all, not even for handlers added
	// with a non-zero resyncPeriod.  For an informer that does
	// resyncs, and for each handler that requests resyncs, that
	// informer develops a nominal resync period that is no shorter
	// than the requested period but may be longer.  The actual time
	// between any two resyncs may be longer than the nominal period
	// because the implementation takes time to do work and there may
	// be competing load and scheduling noise.
	AddEventHandlerWithResyncPeriod(handler ResourceEventHandler, resyncPeriod time.Duration)
	// GetStore returns the informer's local cache as a Store.
	GetStore() Store
	// GetController is deprecated, it does nothing useful
	GetController() Controller
	// Run starts and runs the shared informer, returning after it stops.
	// The informer will be stopped when stopCh is closed.
	Run(stopCh <-chan struct{})
	// HasSynced returns true if the shared informer's store has been
	// informed by at least one full LIST of the authoritative state
	// of the informer's object collection.  This is unrelated to "resync".
	HasSynced() bool
	// LastSyncResourceVersion is the resource version observed when last synced with the underlying
	// store. The value returned is not synchronized with access to the underlying store and is not
	// thread-safe.
	LastSyncResourceVersion() string

	// The WatchErrorHandler is called whenever ListAndWatch drops the
	// connection with an error. After calling this handler, the informer
	// will backoff and retry.
	//
	// The default implementation looks at the error type and tries to log
	// the error message at an appropriate level.
	//
	// There's only one handler, so if you call this multiple times, last one
	// wins; calling after the informer has been started returns an error.
	//
	// The handler is intended for visibility, not to e.g. pause the consumers.
	// The handler should return quickly - any expensive processing should be
	// offloaded.
	SetWatchErrorHandler(handler WatchErrorHandler) error
}
```

### SharedIndexInformer
SharedIndexInformer 扩展了 SharedInformer 接口，提供了构建索引的能力。
```go
type SharedIndexInformer interface {
	SharedInformer
	// AddIndexers add indexers to the informer before it starts.
	AddIndexers(indexers Indexers) error
	GetIndexer() Indexer
}
```

### ListerWatcher
ListerWatcher 是 Informer 机制中的核心对象之一，其功能是通过 List() 方法从 API Server 中获取某一类型的全量数据，再通过 Watch() 方法监听 API Server 中数据的增量更新。

ListerWatcher 继承自 Lister 和 Watcher 接口，从而使其既能获取全量数据，又能监听增量数据更新。

```go
//staging/src/k8s.io/client-go/tools/cache/listwatch.go
type ListerWatcher interface {
	Lister
	Watcher
}
```

```go
// Lister 接口用于完成全量数据的初始化。
type Lister interface {
	// List should return a list type object; the Items field will be extracted, and the
	// ResourceVersion field will be used to start the watch in the right place.
	List(options metav1.ListOptions) (runtime.Object, error)
}

// Watcher 接口用于监听数据的增量更新。
type Watcher interface {
	// Watch should begin a watch at the specified version.
	Watch(options metav1.ListOptions) (watch.Interface, error)
}


```

### DeltaFIFO
![](.14_informer_images/deltaFiFO_info.png)
DeltaFIFO 是一个生产者-消费者的队列，生产者是 Reflector，消费者是 Pop 函数，
从架构图上可以看出 DeltaFIFO 的数据来源为 Reflector，通过 Pop 操作消费数据，
消费的数据一方面存储到 Indexer 中，另一方面可以通过 Informer 的 handler 进行处理，Informer 的 handler 处理的数据需要与存储在 Indexer 中的数据匹配。

需要注意的是，Pop 的单位是一个 Deltas，而不是 Delta。

### Delta
```go
// staging/src/k8s.io/client-go/tools/cache/delta_fifo.go
type Delta struct {
	Type   DeltaType
	Object interface{}
}
```
```go
// DeltaType 是变化类型 (添加、删除等)
type DeltaType string

const (
  Added DeltaType = "Added" // 增加
  Updated DeltaType = "Updated" // 更新
  Deleted DeltaType = "Deleted" // 删除
  Sync DeltaType = "Sync" // 同步
)
```
Delta 是 DeltaFIFO 存储的类型，它记录了对象发生了什么变化以及变化后对象的状态。如果变更是删除，它会记录对象删除之前的最终状态。

### Deltas
```go
type Deltas []Delta

// Oldest is a convenience function that returns the oldest delta, or
// nil if there are no deltas.
func (d Deltas) Oldest() *Delta {
	if len(d) > 0 {
		return &d[0]
	}
	return nil
}

// Newest is a convenience function that returns the newest delta, or
// nil if there are no deltas.
func (d Deltas) Newest() *Delta {
  if n := len(d); n > 0 {
    return &d[n-1]
  }
}
```
Deltas 保存了对象状态的变更（Add/Delete/Update）信息（如 Pod 的删除添加等），Deltas 缓存了针对相同对象的多个状态变更信息，
如 Pod 的 Deltas[0]可能更新了标签，Deltas[1]可能删除了该 Pod。最老的状态变更信息为 Oldest()，最新的状态变更信息为 Newest()，
使用中，获取 DeltaFIFO 中对象的 key 以及获取 DeltaFIFO 都以最新状态为准。

最旧的 delta 在索引0位置，最新的 delta 在最后一个索引位置。

### ResourceEventHandler
当经过List & Watch得到事件时，接下来的实际响应工作就交由ResourceEventHandler来进行，这个Interface定义如下：

```go
type ResourceEventHandler interface {
  // 添加对象回调函数
  OnAdd(obj interface{})
  // 更新对象回调函数
  OnUpdate(oldObj, newObj interface{})
  // 删除对象回调函数
  OnDelete(obj interface{})
}
```
当事件到来时，Informer根据事件的类型（添加/更新/删除资源对象）进行判断，将事件分发给绑定的EventHandler，
即分别调用对应的handle方法（OnAdd/OnUpdate/OnDelete），最后EventHandler将事件发送给Workqueue。


### Indexer
![](.14_informer_images/indexer_info.png)
Indexer在Store基础上扩展了索引能力，就好比给数据库添加的索引，以便查询更快，那么肯定需要有个结构来保存索引。
典型的索引用例是基于对象标签创建索引。 Indexer可以根据多个索引函数维护索引。Indexer使用线程安全的数据存储来存储对象及其键。 
在Store中定义了一个名为MetaNamespaceKeyFunc 的默认函数，该函数生成对象的键作为该对象的<namespace> / <name>组合