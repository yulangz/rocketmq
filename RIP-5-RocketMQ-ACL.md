# ACL Design


## 设计目标
为 Apache RocketMQ 提供权限访问控制

## 总体流程
客户端通过 RPCHook 注入AccessKey，以及利用 SecretKey 签名得到的签名；
服务端通过 RPCHook，对签名进行校验，并对 AccessKey 所拥有的权限进行校验，校验不过，抛出异常；

## 接口设计

参考 `AccessResource` 和 `AccessValidator`

## 权限定义


| 权限 | 含义 |
| --- | --- |
| DENY | 拒绝 |
| ANY | PUB 或者 SUB 权限 |
| PUB | 发送权限 |
| SUB | 订阅权限 |

参考 `Permission` 方法。

## 权限解析

对请求进行解析，拿到需要鉴权的字段。

主要包括：
1.AccessKey，类似于用户名，代指用户主体，权限数据与之对应；
2.Signature，客户根据 SecretKey 签名得到的串，服务端再用SecretKey进行签名验证；
3.ResourceMap, 各个资源以及所需的权限；Topic 的资源名就是 Topic 名字，ConsumerGroup 的资源名就是 %RETRY% + ConsumerGroup 名字；

参考 `PlainAccessValidator` 的 `parse` 方法。 


## 权限校验

校验逻辑：
1. 检查是否命中全局 IP 白名单；如果是，则认为校验通过；否则走 2；
2. 检查是否命中用户 IP 白名单；如果是，则认为校验通过；否则走 3；
3. 校验签名，校验不通过，抛出异常；校验通过，则走 4；
4. 对用户请求所需的权限 和 用户所拥有的权限进行校验；不通过，抛出异常； 

用户所需权限的校验需要注意已下内容：
1.特殊的请求例如 UPDATE_AND_CREATE_TOPIC 等，只能由 admin 账户进行操作；
2.对于某个资源，如果有显性配置权限，则采用配置的权限；如果没有显性配置权限，则采用默认的权限；

参考 `PlainPermissionLoader` 的 `validate` 方法。

## 权限存储

默认实现是基于 yml 的文件存储。

热感知：
检测对权限文件的改动，并自动刷新权限数据。

数据格式：

| 字段 | 取值 | 含义 |
| --- | --- | --- |
| globalWhiteRemoteAddresses | \*;192.168.\*.\*;192.168.0.1 | 全局IP白名单 |
| accessKey | 字符串 | Access Key |
| secretKey | 字符串 | Secret Key |
| whiteRemoteAddress | \*;192.168.\*.\*;192.168.0.1 | 用户IP白名单 |
| admin | true;false | 是否管理员账户 |
| defaultTopicPerm | DENY;PUB;SUB;PUB\|SUB | 默认的Topic权限 |
| defaultGroupPerm | DENY;PUB;SUB;PUB\|SUB | 默认的ConsumerGroup权限 |
| topicPerms | topic=权限 | 各个Topic的权限 |
| groupPerms | group=权限 | 各个ConsumerGroup的权限 |

参考 `transport.yml`


