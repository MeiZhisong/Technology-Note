

# RPC

RPC（Remote Procedure Call）—远程过程调用，它是一种通过网络从远程计算机程序上请求服务，而不需要了解底层网络技术的协议。也就是说两台服务器A，B，一个应用部署在A服务器上，想要调用B服务器上应用提供的方法，由于不在一个内存空间，不能直接调用，需要通过网络来表达调用的语义和传达调用的数据。



**RPC工作原理**

 RPC的设计由Client，Client stub，Network ，Server stub，Server构成。 其中Client就是用来调用服务的，Cient stub是用来把调用的方法和参数序列化的（因为要在网络中传输，必须要把对象转变成字节），Network用来传输这些信息到Server stub， Server stub用来把这些信息反序列化的，Server就是服务的提供者，最终调用的就是Server提供的方法。



 ![RPC工作原理](https://dubbo.apache.org/imgs/blog/rpc/rpc-work-principle.png)





1. Client像调用本地服务似的调用远程服务；
2. Client stub接收到调用后，将方法、参数序列化
3. 客户端通过sockets将消息发送到服务端
4. Server stub 收到消息后进行解码（将消息对象反序列化）
5. Server stub 根据解码结果调用本地的服务
6. 本地服务执行(对于服务端来说是本地执行)并将结果返回给Server stub
7. Server stub将返回结果打包成消息（将结果消息对象序列化）
8. 服务端通过sockets将消息发送到客户端
9. Client stub接收到结果消息，并进行解码（将结果消息反序列化）
10. 客户端得到最终结果。



RPC 调用分以下两种：

1. 同步调用：客户方等待调用执行完成并返回结果。
2. 异步调用：客户方调用后不用等待执行结果返回，但依然可以通过回调通知等方式获取返回结果。若客户方不关心调用返回结果，则变成单向异步调用，单向调用不用返回结果。

异步和同步的区分在于是否等待服务端执行完成并返回结果。





**RPC 能干什么？**

RPC 的主要功能目标是让构建分布式计算（应用）更容易，在提供强大的远程调用能力时不损失本地调用的语义简洁性。为实现该目标，RPC 框架需提供一种透明调用机制，让使用者不必显式的区分本地调用和远程调用，在之前给出的一种实现结构，基于 stub 的结构来实现。

- 可以做到分布式，现代化的微服务
- 部署灵活
- 解耦服务
- 扩展性强

RPC的目的是让你在本地调用远程的方法，而对你来说这个调用是透明的，你并不知道这个调用的方法是部署哪里。通过RPC能解耦服务，这才是使用RPC的真正目的。







# 负载均衡

​	将网络请求，或者其他形式的负载“均摊”到不同的机器上。避免集群中部分服务器压力过大，而另一些服务器比较空闲的情况。通过负载均衡，可以让每台服务器获取到适合自己处理能力的负载。在为高负载服务器分流的同时，还可以避免资源浪费，一举两得。负载均衡可分为软件负载均衡和硬件负载均衡。



 ![image-20210714143144503](/Users/meizhisong/Library/Application Support/typora-user-images/image-20210714143144503.png)

 





**LoadBalance**

```java
@SPI("random")
public interface LoadBalance {

    @Adaptive("loadbalance")
    <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException;

}
```



**AbstractLoadBalance**

```java
@Override
public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // 判断Invoker列表是否为空
    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    // 如果Invoker列表只有一个Invoker对象，直接返回该Invoker对象
    if (invokers.size() == 1) {
        return invokers.get(0);
    }
    // 抽象方法 
    return doSelect(invokers, url, invocation);
}
```





**1、RandomLoadBalance**

即加权随机策略。

<font color=#999AAA>假设我们有一组服务器 servers = [A, B, C]，他们对应的权重为 weights = [5, 3, 2]，权重总和为10。现在把这些权重值平铺在一维坐标值上，[0, 5) 区间属于服务器 A，[5, 8) 区间属于服务器 B，[8, 10) 区间属于服务器 C。接下来通过随机数生成器生成一个范围在 [0, 10) 之间的随机数，然后计算这个随机数会落到哪个区间上。比如数字3会落到服务器 A 对应的区间上，此时返回服务器 A 即可。权重越大的机器，在坐标轴上对应的区间范围就越大，因此随机数生成器生成的数字就会有更大的概率落到此区间内。只要随机数生成器产生的随机数分布性很好，在经过多次选择后，每个服务器被选中的次数比例接近其权重比例。比如，经过一万次选择后，服务器 A 被选中的次数大约为5000次，服务器 B 被选中的次数约为3000次，服务器 C 被选中的次数约为2000次。</font>



```JAVA
@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  // Invoker的数量
  int length = invokers.size();
  // 所有的Invoker的权重是否相同
  boolean sameWeight = true;
  int[] weights = new int[length];
  // Invoker列表中第一个Invoker的权重
  int firstWeight = getWeight(invokers.get(0), invocation);
  weights[0] = firstWeight;
  // 所有Invoker的权重总和
  int totalWeight = firstWeight;
  for (int i = 1; i < length; i++) {
    // 获取Invoker对象对应的权重值
    int weight = getWeight(invokers.get(i), invocation);
    weights[i] = weight;
    totalWeight += weight;
    // 判断所有Invoker的权重是否是完全相同的
    if (sameWeight && weight != firstWeight) {
      sameWeight = false;
    }
  }
  // 如果所有Invoker的权重总和大于0，并且权重不是全部相同
  if (totalWeight > 0 && !sameWeight) {
    // 在[0, totalWeight]之间生成生成随机整数
    int offset = ThreadLocalRandom.current().nextInt(totalWeight);
    for (int i = 0; i < length; i++) {
      offset -= weights[i];
      // 计算随机数落到哪个区间
      if (offset < 0) {
        // 获取对应区间的Invoker对象
        return invokers.get(i);
      }
    }
  }
  // 如果所有Invoker权重相同或者权重总和等于0，
  // 在[0, Invoker)之间生成随机整数，
  // 获取Invoker列表对应索引位置的Invoker对象
  return invokers.get(ThreadLocalRandom.current().nextInt(length));
}
```



```java
int getWeight(Invoker<?> invoker, Invocation invocation) {
  int weight;
  URL url = invoker.getUrl();
  // 如果服务接口是RegistryService
  if ("org.apache.dubbo.registry.RegistryService".equals(url.getServiceInterface())) {
    // 从 url 中获取权重 registry.weight 配置值
    weight = url.getParameter("registy.weight", "100");
  } else {
    // 从 url 中获取权重 weight 配置值
    weight = url.getMethodParameter(invocation.getMethodName(), "weight", "100");
    if (weight > 0) {
      // 获取服务提供者启动时间戳
      long timestamp = invoker.getUrl().getParameter("timestamp", 0L);
      if (timestamp > 0L) {
        // 计算服务提供者运行时长
        long uptime = System.currentTimeMillis() - timestamp;
        if (uptime < 0) {
          return 1;
        }
        // 获取服务预热时间，默认为10分钟
        int warmup = invoker.getUrl().getParameter("warmup", "600000");
        // 如果服务运行时间小于预热时间，则重新计算服务权重，即降权
        if (uptime > 0 && uptime < warmup) {
          // 重新计算服务权重
          weight = calculateWarmupWeight((int)uptime, warmup, weight);
        }
      }
    }
  }
  return Math.max(weight, 0);
}
```



```java
static int calculateWarmupWeight(int uptime, int warmup, int weight) {
  	// 计算权重，下面代码逻辑上形似于 (uptime / warmup) * weight。
    // 随着服务运行时间 uptime 增大，权重计算值 ww 会慢慢接近配置值 weight
    int ww = (int) ( uptime / ((float) warmup / weight));
    return ww < 1 ? 1 : (Math.min(ww, weight));
}
```

该过程主要用于保证当服务运行时长小于服务预热时间时，对服务进行降权，避免让服务在启动之初就处于高负载状态。服务预热是一个优化手段，与此类似的还有 JVM 预热。主要目的是让服务启动后“低功率”运行一段时间，使其效率慢慢提升至最佳状态。



**2、LeastActiveLoadBalance**

即最小活跃数策略。



<font color=#999AAA>活跃调用数越小，表明该服务提供者效率越高，单位时间内可处理更多的请求。此时应优先将请求分配给该服务提供者。在具体实现中，每个服务提供者对应一个活跃数 active。初始情况下，所有服务提供者活跃数均为0。每收到一个请求，活跃数加1，完成请求后则将活跃数减1。在服务运行一段时间后，性能好的服务提供者处理请求的速度更快，因此活跃数下降的也越快，此时这样的服务提供者能够优先获取到新的服务请求、这就是最小活跃数负载均衡算法的基本思想。除了最小活跃数，LeastActiveLoadBalance 在实现上还引入了权重值。所以准确的来说，LeastActiveLoadBalance 是基于加权最小活跃数算法实现的。举个例子说明一下，在一个服务提供者集群中，有两个性能优异的服务提供者。某一时刻它们的活跃数相同，此时 Dubbo 会根据它们的权重去分配请求，权重越大，获取到新请求的概率就越大。如果两个服务提供者权重相同，此时随机选择一个即可。</font>



```java
@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
    // Number of invokers
    int length = invokers.size();
    // The least active value of all invokers
    int leastActive = -1;
    // 具有最小活跃数的服务提供者的数量
    int leastCount = 0;
    // 保存每一个具有最小活跃数的Invoker
    int[] leastIndexes = new int[length];
    // 保存每一个服务提供者
    int[] weights = new int[length];
    // The sum of the warmup weights of all the least active invokers
    int totalWeight = 0;
    // 保存第一个具有最小活跃数的服务提供者的权重
    int firstWeight = 0;
    // 每一个具有最小活跃数的服务提供者是否具有相同的权重
    boolean sameWeight = true;

    for (int i = 0; i < length; i++) {
        Invoker<T> invoker = invokers.get(i);
        // 获取Invoker对应的活跃数
        int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
        // 获取权重
        int afterWarmup = getWeight(invoker, invocation);
        weights[i] = afterWarmup;
        // 如果发现了更小的活跃数
        if (leastActive == -1 || active < leastActive) {
            // 重置最小活跃数
            leastActive = active;
            // 重置leaseCount为1
            leastCount = 1;
            leastIndexes[0] = i;
            totalWeight = afterWarmup;
            firstWeight = afterWarmup;
            sameWeight = true;
        // 当前 Invoker 的活跃数 active 与最小活跃数 leastActive 相同 
        } else if (active == leastActive) {
            // 在 leastIndexs 中记录下当前 Invoker 在 invokers 集合中的下标
            leastIndexes[leastCount++] = i;
            // 累加权重
            totalWeight += afterWarmup;
            // 检测当前 Invoker 的权重与 firstWeight 是否相等，
            // 不相等则将 sameWeight 置为 false
            if (sameWeight && i > 0
                    && afterWarmup != firstWeight) {
                sameWeight = false;
            }
        }
    }
    // 当只有一个 Invoker 具有最小活跃数，此时直接返回该 Invoker 即可
    if (leastCount == 1) {
        return invokers.get(leastIndexes[0]);
    }
    // 有多个 Invoker 具有相同的最小活跃数，但它们之间的权重不同
    if (!sameWeight && totalWeight > 0) {
        // 随机生成一个 [0, totalWeight) 之间的数字
        int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
        // 循环让随机数减去具有最小活跃数的 Invoker 的权重值，
        // 当 offset 小于等于0时，返回相应的 Invoker
        for (int i = 0; i < leastCount; i++) {
            int leastIndex = leastIndexes[i];
            // 获取权重值，并让随机数减去权重值
            offsetWeight -= weights[leastIndex];
            if (offsetWeight < 0) {
                return invokers.get(leastIndex);
            }
        }
    }
    // 如果权重相同或权重为0时，随机返回一个 Invoker
    return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
}
```

 

 ![image-20210621175122166](/Users/meizhisong/Library/Application Support/typora-user-images/image-20210621175122166.png)





**3、ConsistentHashLoadBalance**



即一致性hash策略。



Dubbo的一致性Hash算法也可以分为两步：

**1、映射Provider至Hash值区间中（实际中映射的是Invoker）；**

**2、映射请求，然后找到大于请求Hash值的第一个Invoker。**



  ![img](https://dubbo.apache.org/imgs/dev/consistent-hash-invoker.jpg)



引入虚拟节点，让 Invoker 在圆环上分散开来，避免数据倾斜问题。

所谓数据倾斜是指，由于节点不够分散，导致大量请求落到了同一个节点上，而其他节点只会接收到了少量请求的情况。



```java
@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  String methodName = RpcUtils.getMethodName(invocation);
  String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
  // 获取 invokers 原始的 hashcode
  int invokersHashCode = invokers.hashCode();
  ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
  // 如果 invokers 是一个新的 List 对象，意味着服务提供者数量发生了变化，可能新增也可能减少了。
  // 此时 selector.identityHashCode != identityHashCode 条件成立
  if (selector == null || selector.identityHashCode != invokersHashCode) {
    // 创建新的 ConsistentHashSelector
    selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, invokersHashCode));
    selector = (ConsistentHashSelector<T>) selectors.get(key);
  }
  // 调用 ConsistentHashSelector 的 select 方法选择 Invoker
  return selector.select(invocation);
}
```

检测 invokers 列表是不是变动过，以及创建 ConsistentHashSelector。

调用 ConsistentHashSelector 的 select 方法执行负载均衡逻辑。



**ConsistentHashSelector**

```java
private static final class ConsistentHashSelector<T> {

  	// 使用 TreeMap 存储 Invoker 虚拟节点
    private final TreeMap<Long, Invoker<T>> virtualInvokers;

    private final int replicaNumber;

    private final int identityHashCode;

    private final int[] argumentIndex;

    ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
        this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
        this.identityHashCode = identityHashCode;
        URL url = invokers.get(0).getUrl();
      	// 获取虚拟节点数，默认为160
        this.replicaNumber = url.getMethodParameter(methodName, HASH_NODES, 160);
      	// 获取参与 hash 计算的参数下标值，默认对第一个参数进行 hash 运算
        String[] index = COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, HASH_ARGUMENTS, "0"));
        argumentIndex = new int[index.length];
        for (int i = 0; i < index.length; i++) {
            argumentIndex[i] = Integer.parseInt(index[i]);
        }
        for (Invoker<T> invoker : invokers) {
            String address = invoker.getUrl().getAddress();
            for (int i = 0; i < replicaNumber / 4; i++) {
              	// 对 address + i 进行 md5 运算，得到一个长度为16的字节数组
                byte[] digest = md5(address + i);
                // 对 digest 部分字节进行4次 hash 运算，得到四个不同的 long 型正整数
                for (int h = 0; h < 4; h++) {
                  	// h = 0 时，取 digest 中下标为 0 ~ 3 的4个字节进行位运算
                    // h = 1 时，取 digest 中下标为 4 ~ 7 的4个字节进行位运算
                    // h = 2, h = 3 时过程同上
                    long m = hash(digest, h);
                  	// 将 hash 到 invoker 的映射关系存储到 virtualInvokers 中，
                    // virtualInvokers 需要提供高效的查询操作，因此选用 TreeMap 作为存储结构
                    virtualInvokers.put(m, invoker);
                }
            }
        }
    }
}
```

从配置中获取虚拟节点数以及参与 hash 计算的参数下标，默认情况下只使用第一个参数进行 hash。

计算虚拟节点 hash 值，并将虚拟节点存储到 TreeMap 中。



```java
public Invoker<T> select(Invocation invocation) {
  // 将参数转为 key
  String key = toKey(invocation.getArguments());
  // 对参数 key 进行 md5 运算
  byte[] digest = md5(key);
  // 取 digest 数组的前四个字节进行 hash 运算，再将 hash 值传给 selectForKey 方法，
  // 寻找合适的 Invoker
  return selectForKey(hash(digest, 0));
}

private Invoker<T> selectForKey(long hash) {
    // 到 TreeMap 中查找第一个节点值大于或等于当前 hash 的 Invoker
    Map.Entry<Long, Invoker<T>> entry = virtualInvokers.tailMap(hash, true).firstEntry();
    // 如果 hash 大于 Invoker 在圆环上最大的位置，此时 entry = null，
    // 需要将 TreeMap 的头节点赋值给 entry
    if (entry == null) {
        entry = virtualInvokers.firstEntry();
    }
    // 返回 Invoker
    return entry.getValue();
}
```

首先是对参数进行 md5 以及 hash 运算，得到一个 hash 值。

再拿这个值到 TreeMap 中查找第一个节点值大于或等于当前 hash 的 Invoker。





**4、RoundRobinLoadBalance**



即加权轮询策略。



所谓轮询是指将请求轮流分配给每台服务器。轮询是一种无状态负载均衡算法，实现简单，适用于每台服务器性能相近的场景下。经过加权后，每台服务器能够得到的请求数比例，接近或等于他们的权重比。



使用服务器 [A, B, C] 对应权重 [5, 1, 1] 的例子说明，现在有7个请求依次进入负载均衡逻辑，选择过程如下：

| 请求编号 | currentWeight 数组 | 选择结果 | 减去权重总和后的 currentWeight 数组 |
| -------- | ------------------ | -------- | ----------------------------------- |
| 1        | [5, 1, 1]          | A        | [-2, 1, 1]                          |
| 2        | [3, 2, 2]          | A        | [-4, 2, 2]                          |
| 3        | [1, 3, 3]          | B        | [1, -4, 3]                          |
| 4        | [6, -3, 4]         | A        | [-1, -3, 4]                         |
| 5        | [4, -2, 5]         | C        | [4, -2, -2]                         |
| 6        | [9, -1, -1]        | A        | [2, -1, -1]                         |
| 7        | [7, 0, 0]          | A        | [0, 0, 0]                           |



```java
@Override
protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
  	// key = 全限定类名 + "." + 方法名，比如 com.xxx.DemoService.sayHello
    String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
  
  	// 最外层为服务类名 + 方法名，第二层为 url 到 WeightedRoundRobin 的映射关系。
    // 这里我们可以将 url 看成是服务提供者的 id
  
  	// 获取 url 到 WeightedRoundRobin 映射表，如果为空，则创建一个新的
    ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.computeIfAbsent(key, k -> new ConcurrentHashMap<>());
  	// 权重和
    int totalWeight = 0;
  	// 当前最大的权重
    long maxCurrent = Long.MIN_VALUE;
    long now = System.currentTimeMillis();
    Invoker<T> selectedInvoker = null;
    WeightedRoundRobin selectedWRR = null;
    // 遍历 Invoker 列表
    for (Invoker<T> invoker : invokers) {
      	// 获取 url 唯一标识 identifyString
        String identifyString = invoker.getUrl().toIdentityString();
        // 获取 Invoker 对应的权重
        int weight = getWeight(invoker, invocation);
        WeightedRoundRobin weightedRoundRobin = map.get(identifyString);

      	// 检测当前 Invoker 是否有对应的 WeightedRoundRobin，没有则创建
        if (weightedRoundRobin == null) {
            weightedRoundRobin = new WeightedRoundRobin();
          	// 设置 Invoker 权重
            weightedRoundRobin.setWeight(weight);
          	// 存储 url 唯一标识 identifyString 到 weightedRoundRobin 的映射关系
            map.putIfAbsent(identifyString, weightedRoundRobin);
            weightedRoundRobin = map.get(identifyString);
        }
        // 如果权重发生了变化
        if (weight != weightedRoundRobin.getWeight()) {
            // 更新新的权重
            weightedRoundRobin.setWeight(weight);
        }
      	// CAS方式更新权重 等价于 current += weight
        long cur = weightedRoundRobin.increaseCurrent();
      	// 设置更新时间
        weightedRoundRobin.setLastUpdate(now);
        // 找出最大的当前权重值
        if (cur > maxCurrent) {
            maxCurrent = cur;
          	// 将具有最大 current 权重的 Invoker 赋值给 selectedInvoker
            selectedInvoker = invoker;
          	// 将 Invoker 对应的 weightedRoundRobin 赋值给 selectedWRR，留作后用
            selectedWRR = weightedRoundRobin;
        }
        // 将当前权重累加到权重和
        totalWeight += weight;
    }
  
  	// 对 <identifyString, WeightedRoundRobin> 进行检查，过滤掉长时间未被更新的节点。
  	// 该节点可能挂了，invokers 中不包含该节点，所以该节点的 lastUpdate 长时间无法被更新。
  	// 若未更新时长超过阈值后，就会被移除掉，默认阈值为60秒。
    if (!updateLock.get() && invokers.size() != map.size()) {
        if (updateLock.compareAndSet(false, true)) {
            try {
                ConcurrentMap<String, WeightedRoundRobin> newMap = new ConcurrentHashMap<>(map);
              	// 剔除更新时间超过1分钟的元素
                newMap.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > 60000);
                methodWeightMap.put(key, newMap);
            } finally {
                updateLock.set(false);
            }
        }
    }
    if (selectedInvoker != null) {
      	// 让 current 减去权重总和，等价于 current -= totalWeight
        selectedWRR.sel(totalWeight);
        // 返回具有最大 current 的 Invoker
        return selectedInvoker;
    }
    // should not happen here
    return invokers.get(0);
}
```

