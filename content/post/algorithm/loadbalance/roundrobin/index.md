---
title: 平滑的加权轮询算法
date: 2023-07-24
description: 加权轮询算法是负载均衡算法中最常用的算法之一，本文介绍的是由 nginx 实现的一种平滑的加权轮询算法，能够将请求根据权重均匀地轮询分摊到每个节点上
image: cover.jpg
tags:
  - LoadBalancer
  - LoadBalance
  - RoundRobin
categories:
  - note
---

nginx 中使用 upstream 标识转发的上游服务的服务列表，如下：

```conf
upstream example {
    server 192.168.21.1:8080;
    server 192.168.21.2:8080;
}
```

默认使用的负载均衡策略是轮询，当服务节点的承载能力不一样时，nginx 可以支持使用 `weight` 属性确定节点的负载权重，如下所示：

```conf
upstream cluster {
	server 192.168.21.1:8080 weight=2;
	server 192.168.21.2:8080 weight=1;
	server 192.168.21.3:8080 weight=3;
}
```

按照常规的思路，每 6 个请求一轮回，前 2 次转到 `192.168.21.1:8080` ，第 3 次转到 `192.168.21.2:8080` ，最后 3 次转到 `192.168.21.3:8080` 即可

但是这样的加权轮询算法有一个问题，就是不够“平滑”，假设权重的列表是类似 [10, 2, 1] 这种形式，那么前十次请求将会全部转发到第一个节点，要是能把后面两个节点穿插到这些请求中就好了

nginx 就使用一种“平滑”的加权轮询算法实现了这一特点，在上面的例子中，我们假设

- 192.168.21.1:8080：A
- 192.168.21.2:8080：B
- 192.168.21.3:8080：C

对于普通的加权轮询算法，其 6 次请求的负载列表如下：

[A, A, B, C, C, C]

而使用 nginx 的加权轮询算法，得到的负载列表则如下所示：

[C, A, B, C, A, C]

可以看到 nginx 的加权轮询算法得到的负载列表更加的均匀，有效地将所有节点根据权重均匀的分摊开来，下面让我们看看 nginx 是如何实现的

## 原理

### 参数

在 nginx 的加权轮询算法中，有 3 个重要的参数：

- weight：配置文件中指定的权重，这个值是固定不变的
- effective_weight：节点的有效权重，其初始值是 weight。在释放后端的过程中，如果与节点的通信发生错误，就减小 effective_weight；在后续请求中，选取节点时再逐步增加 effective_weight 直至恢复到 weight。这个参数的作用主要就是当节点发生错误时，减少其权重，减少分摊到此节点的请求
- current_weight：节点当前的权重，初始值为 0，会在选取过程中动态调整。选取过程中，遍历所有节点，使得每个节点的 current_weight 等于原 current_weight 加上其 effective_weight，同时累加所有节点的 effective_weight 为 total，然后选取 current_weight 最大的节点作为最终的选中节点，并将其 current_weight 等于原 current_weight 减去 total（没有被选中的节点无需减去 total）

### 举个栗子

这么说可能有些抽象，用一个列表来描述可能更加形象一些，还是上面 [2, 1, 3] 的权重列表，依旧以 [A, B, C] 代替节点，其负载过程如下所示：

| 请求序列 | 选定之前的 current_weight 列表 | 选中节点 | 选定之后的 current_weight 列表 |
| -------- | ------------------------------ | -------- | ------------------------------ |
| 1        | [2, 1, 3]                      | C        | [2, 1, -3]                     |
| 2        | [4, 2, 0]                      | A        | [-2, 2, 0]                     |
| 3        | [0, 3, 3]                      | B        | [0, -3, 3]                     |
| 4        | [2, -2, 6]                     | C        | [2, -2, 0]                     |
| 5        | [4, -1, 3]                     | A        | [-2, -1, 3]                    |
| 6        | [0, 0, 6]                      | C        | [0, 0, 0]                      |
| 7        | [2, 1, 3]                      | C        | [2, 1, -3]                     |

可以看到，其前 6 次负载列表为：[C, A, B, C, A, C]，且在第 6 次请求后，current_weight 列表恢复到了初始的 [0, 0, 0] 状态，也就是说其负载列表将会一直以 [C, A, B, C, A, C] 的形式循环下去

## 实现

下面以 Java 为例，实现这一负载均衡算法

### 节点类

首先定义一个简单的节点类：

```java
public class Instance implements Serializable {
    @Serial
    private static final long serialVersionUID = 535926252786888744L;

    private String host;
    private Integer port;
    private Integer weight;
    private Integer effectiveWeight;
    private Integer currentWeight;

    public Instance() {
    }

    public Instance(String host, Integer port, Integer weight) {
        this.host = host;
        this.port = port;
        this.weight = weight;
        this.effectiveWeight = weight;
        this.currentWeight = 0;
    }

    public String getHost() {
        return host;
    }

    public void setHost(String host) {
        this.host = host;
    }

    public Integer getPort() {
        return port;
    }

    public void setPort(Integer port) {
        this.port = port;
    }

    public Integer getWeight() {
        return weight;
    }

    public void setWeight(Integer weight) {
        this.weight = weight;
    }

    public Integer getEffectiveWeight() {
        return effectiveWeight;
    }

    public void setEffectiveWeight(Integer effectiveWeight) {
        this.effectiveWeight = effectiveWeight;
    }

    public Integer getCurrentWeight() {
        return currentWeight;
    }

    public void setCurrentWeight(Integer currentWeight) {
        this.currentWeight = currentWeight;
    }

    @Override
    public String toString() {
        return "Instance{" +
                "host='" + host + '\'' +
                ", port=" + port +
                ", weight=" + weight +
                '}';
    }
}
```

这个节点类除了包含最基本的 `host`、`port`、`weight` 属性外，还有两个属性，即 `effectiveWeight` 和 `currentWeight`，用来表示节点的有效权重和当前权重

### 负载均衡器

接着定义一个 `LoadBalancer` 接口类：

```java
public interface LoadBalancer {
    Instance choose();
}
```

编写加权轮询负载均衡器实现这个接口：

```java
public class RoundRobinLoadBalancer implements LoadBalancer {
    private final List<Instance> instances;

    public RoundRobinLoadBalancer(List<Instance> instances) {
        this.instances = instances;
    }

    @Override
    public Instance choose() {
        int total = 0;
        for (Instance instance : instances) {
            int effectiveWeight = instance.getEffectiveWeight();
            int currentWeight = instance.getCurrentWeight();
            currentWeight += effectiveWeight;
            instance.setCurrentWeight(currentWeight);
            total += effectiveWeight;
        }
        int max = 0;
        int index = 0;
        for (int i = 0; i < instances.size(); i++) {
            int currentWeight = instances.get(i).getCurrentWeight();
            if (currentWeight > max) {
                max = currentWeight;
                index = i;
            }
        }
        Instance selectedInstance = instances.get(index);
        int selectedWeight = selectedInstance.getCurrentWeight();
        int currentWeight = selectedWeight - total;
        selectedInstance.setCurrentWeight(currentWeight);
        return selectedInstance;
    }
}
```

### 测试

测试是否运行正确：

```java
public class RoundRobinLoadBalancerTest {
    @Test
    public void test() {
        List<Instance> instances = List.of(
                new Instance("192.168.21.1", 8080, 2),
                new Instance("192.168.21.2", 8080, 1),
                new Instance("192.168.21.3", 8080, 3)
        );
        LoadBalancer loadBalancer = new RoundRobinLoadBalancer(instances);
        List<String> expectedHosts = List.of(
                "192.168.21.3",
                "192.168.21.1",
                "192.168.21.2",
                "192.168.21.3",
                "192.168.21.1",
                "192.168.21.3"
        );
        for (int i = 0; i < 18; i++) {
            Instance instance = loadBalancer.choose();
            String actual = instance.getHost();
            int index = i % 6;
            String expected = expectedHosts.get(index);
            Assertions.assertEquals(expected, actual);
        }
    }
}
```

如期运行

> Cover from [Sergio Zhukov](https://unsplash.com/@opohmelka) on [unsplash](https://unsplash.com/)
