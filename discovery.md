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

# APISIX集成Nacos作为服务发现中心

* [**摘要**](#摘要)
* [**如何在APISIX中使用Nacos**](#如何在APISIX中使用Nacos)
    * [**接入**](#接入)
    * [**upstream 配置**](#upstream-配置)

## 摘要

常见的服务注册发现中心有：Eureka, Etcd, Consul, Zookeeper, Nacos等

目前APISIX官方支持 Eureka 和基于 DNS 的服务注册发现，如 Consul 等，并实现了Eureka的接入代码。参看 [APISIX服务发现](https://github.com/apache/apisix/blob/master/docs/en/latest/discovery.md)

本项目描述了在APISIX中如何使用Nacos作为注册中心

## 如何在APISIX中使用Nacos

### 接入

首先，在 `apisix/discovery/` 目录中添加 [`nacos.lua`](./apisix/discovery/nacos.lua);

然后，在 `conf/config.yaml` 增加如下格式的配置，参看[`config.yaml`](./conf/config.yaml);

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

### upstream 配置

APISIX是通过 `upstream.discovery_type`选择使用的服务发现， `upstream.service_name` 与注册中心的服务名进行关联。例如执行如下命令：

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
**服务名称说明**

在Nacos中定位一个服务是从namespace、group和service三个维度进行的。所以service_name里面可以同时传入三个维度的值，格式：`namespaceId:groupName:serviceName`

Nacos的默认namespaceId是public，默认groupName是DEFAULT_GROUP；

当service_name中，仅有一个`:`时，表示`namespaceId:serviceName`，此时groupName是DEFAULT_GROUP

当service_name中，没有`:`时，表示`serviceName`，此时namespaceId是public，groupName是DEFAULT_GROUP

**注意**：配置 `upstream.service_name` 后 `upstream.nodes` 将不再生效，而是使用从注册中心的数据来替换，即使注册中心的数据是空的。
