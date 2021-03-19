<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->
<!-- [English](../discovery.md) -->

# 集成服务发现注册中心

* [**摘要**](#摘要)
* [**当前支持的注册中心**](#当前支持的注册中心)
* [**如何扩展注册中心**](#如何扩展注册中心)
    * [**基本步骤**](#基本步骤)
    * [**以 Nacos 举例**](#以-Nacos-举例)
        * [**实现 nacos.lua**](#实现-nacoslua)
        * [**Nacos 与 APISIX 之间数据转换逻辑**](#Nacos-与-APISIX-之间数据转换逻辑)
* [**注册中心配置**](#注册中心配置)
    * [**初始化服务发现**](#初始化服务发现)
    * [**Nacos 的配置**](#Nacos-的配置)
* [**upstream 配置**](#upstream-配置)

## 摘要

当业务量发生变化时，需要对上游服务进行扩缩容，或者因服务器硬件故障需要更换服务器。如果网关是通过配置来维护上游服务信息，在微服务架构模式下，其带来的维护成本可想而知。再者因不能及时更新这些信息，也会对业务带来一定的影响，还有人为误操作带来的影响也不可忽视，所以网关非常必要通过服务注册中心动态获取最新的服务实例信息。架构图如下所示：

![](../images/discovery-cn.png)

1. 服务启动时将自身的一些信息，比如服务名、IP、端口等信息上报到注册中心；各个服务与注册中心使用一定机制（例如心跳）通信，如果注册中心与服务长时间无法通信，就会注销该实例；当服务下线时，会删除注册中心的实例信息；
2. 网关会准实时地从注册中心获取服务实例信息；
3. 当用户通过网关请求服务时，网关从注册中心获取的实例列表中选择一个进行代理；

常见的注册中心：Eureka, Etcd, Consul, Nacos, Zookeeper等

## 当前支持的注册中心

目前APISIX官方支持 Eureka 和基于 DNS 的服务注册发现，如 Consul 等。

基于 DNS 的服务注册发现见 [基于 DNS 的服务支持发现](../dns.md#service-discovery-via-dns)。

## 如何扩展注册中心，例如使用Nacos？

### 基本步骤

APISIX 要扩展注册中心其实是件非常容易的事情，其基本步骤如下：

1. 在 `apisix/discovery/` 目录中添加注册中心客户端的实现；
2. 实现用于初始化的 `_M.init_worker()` 函数以及用于获取服务实例节点列表的 `_M.nodes(service_name)` 函数；
3. 将注册中心数据转换为 APISIX 格式的数据；

### 以 Nacos 举例

#### 实现 nacos.lua

首先在 `apisix/discovery/` 目录中添加 [`nacos.lua`](../../apisix/discovery/nacos.lua);

然后在 `nacos.lua` 实现用于初始化的 `init_worker` 函数以及用于获取服务实例节点列表的 `nodes` 函数即可：

  ```lua
  local _M = {
      version = 0.1,
  }


  function _M.nodes(service_name)
      ... ...
  end


  function _M.init_worker()
      ... ...
  end


  return _M
  ```

#### Nacos 与 APISIX 之间数据转换逻辑

APISIX是通过 `upstream.nodes` 来配置上游服务的，所以使用注册中心后，通过注册中心获取服务的所有 node 后，赋值给 `upstream.nodes` 来达到相同的效果。那么 APISIX 是怎么将 Nacos 的数据转成 node 的呢？ 

首先，要获取Nacos的accessToken，然后获取instance列表，并解析为APISIX的nodes。

APISIX nodes 的格式如下：

```json
[
  {
    "host" : "192.168.1.100",
    "port" : 8761,
    "weight" : 100,
    "metadata" : {
      "management.port": "8761",
    }
  }
]
```

## 注册中心配置

### 初始化服务发现

在 `conf/config.yaml` 增加如下格式的配置：

```yaml
discovery:                     
  nacos:
    host:                     
      - "http://127.0.0.1:8848"
    username: nacos_username
    password: nacos_password
    prefix: "/nacos/v1/"
    fetch_interval: 5           
    weight: 100                 
    timeout:
      connect: 2000             
      send: 2000               
      read: 5000               
```

## upstream 配置

APISIX是通过 `upstream.discovery_type`选择使用的服务发现， `upstream.service_name` 与注册中心的服务名进行关联。下面是将 URL 为 "/user/*" 的请求路由到注册中心名为 "USER-SERVICE" 的服务上例子：

```shell
$ curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "uri": "/ping",
    "upstream": {
        "name": "ping",
        "service_name": "dev:DEFAULT_GROUP:ping_demo",
        "type": "roundrobin",
        "discovery_type": "nacos"
    }
}'
```

在nacos中有namespace、group和service三个维度，所以service_name里面可以同时传入三个维度的值`namespaceId:groupName:serviceName`
nacos的默认namespaceId是public，默认groupName是DEFAULT_GROUP；
当service_name中，仅有一个:时，表示`namespaceId:serviceName`，此时groupName是DEFAULT_GROUP
当service_name中，没有:时，表示`serviceName`，此时namespaceId是public，groupName是DEFAULT_GROUP

**注意**：配置 `upstream.service_name` 后 `upstream.nodes` 将不再生效，而是使用从注册中心的数据来替换，即使注册中心的数据是空的。
