# HTTP Log

> 本文原文链接：https://docs.konghq.com/hub/kong-inc/http-log/

将请求和响应日志发送到HTTP服务器。

> 注意：此插件的功能与0.12.0之前的Kong版本捆绑在一起，与此处记录的不同。
有关详细信息，请参阅[CHANGELOG](https://github.com/Kong/kong/blob/master/CHANGELOG.md)。

## 术语

- `plugin`: 在请求被代理到上游API之前或之后，在Kong内部执行操作的插件。
- `Service`: 表示外部 *upstream* API或微服务的Kong实体。
- `Route`: 表示将下游请求映射到上游服务的方法的Kong实体。
- `Consumer`: 代表使用API的开发人员或机器的Kong实体。当使用Kong时，Consumer 仅与Kong通信，其代理对所述上游API的每次调用。
- `Credential`: 与Consumer关联的唯一字符串，也称为API密钥。
- `upstream service`: 这是指位于Kong后面的您自己的 API/service，转发客户端请求。

## 配置

此插件与具有以下协议的请求兼容：

- `http`
- `https`
- `grpc`
- `grpcs`

此插件与无DB模式兼容。

## 在 Service 上启用插件

**使用数据库：**

通过发出以下请求在Service上配置此插件：
```bash
$ curl -X POST http://kong:8001/services/{service}/plugins \
    --data "name=http-log"  \
    --data "config.http_endpoint=http://mockbin.org/bin/:id" \
    --data "config.method=POST" \
    --data "config.timeout=1000" \
    --data "config.keepalive=1000"
```

**不使用数据库：**

通过添加此部分在服务上配置此插件执行声明性配置文件：

```yaml
plugins:
- name: http-log
  service: {service}
  config: 
    http_endpoint: http://mockbin.org/bin/:id
    method: POST
    timeout: 1000
    keepalive: 1000
```
在这两种情况下，`{service}`是此插件配置将定位的Service的`id`或`name`。

## 在 Consumer 上启用插件

**使用数据库：**

您可以使用`http://localhost:8001/plugins`在特定的Consumers上启用此插件:

```bash
$ curl -X POST http://kong:8001/consumers/{consumer}/plugins \
    --data "name=http-log" \
     \
    --data "config.http_endpoint=http://mockbin.org/bin/:id" \
    --data "config.method=POST" \
    --data "config.timeout=1000" \
    --data "config.keepalive=1000"
```

**不使用数据库：**

通过添加此部分在Consumer上配置此插件执行声明性配置文件：

```
plugins:
- name: http-log
  consumer: {consumer}
  config: 
    http_endpoint: http://mockbin.org/bin/:id
    method: POST
    timeout: 1000
    keepalive: 1000
```
在这两种情况下，`{consumer`}都是此插件配置将定位的`Consumer`的`id`或`username`。  
您可以组合`consumer_id`和`service_id` 。 
在同一个请求中，进一步缩小插件的范围。

## 全局插件

- **使用数据库：** 可以使用`http://kong:8001/plugins/`配置所有插件。
- **不使用数据库：** 可以通过`plugins`: 配置所有插件：声明性配置文件中的条目。

与任何 Service ，Route 或 Consumer （或API，如果您使用旧版本的Kong）无关的插件被视为“全局”，并将在每个请求上运行。有关更多信息，请阅读[插件参考](https://docs.konghq.com/latest/admin-api/#add-plugin)和[插件优先级](https://docs.konghq.com/latest/admin-api/#precedence)部分。

## 参数

以下是可在此插件配置中使用的所有参数的列表：

| 参数 | 默认值 | 描述 |
| ---- | ------ | ---- |
| `name` |  |  要使用的插件的名称，在本例中为`http-log`  |
| `service_id` |  | 此插件将定位的 Service 的ID。|
| `route_id` |  |  此插件将定位的 Route 的ID。 |
| `enabled` |  `true` | 是否将应用此插件。  |
| `consumer_id` |  | 此插件将定位的Consumer的id  |
| `config.http_endpoin` | | 将数据将发送到的HTTP端点（包括要使用的协议）。 | 
| `config.method` <br> *optional* | `POST` | 用于将数据发送到http服务器的可选方法，其他受支持的值为`PUT`，`PATCH` |
| `config.timeout` <br> *optional* | `10000` | 向上游服务器发送数据时的可选超时（以毫秒为单位） |
| `config.keepalive` <br> *optional* | `60000` | 一个可选值（以毫秒为单位），用于定义空闲连接在关闭之前的生存时间 |

> 注意：如果`config.http_endpoint`包含用户名和密码（例如`http://bob：password@example.com/logs`），则Kong将在日志请求中自动包括basic-auth `Authorization` 请求头。

## 日志格式

每个请求都将分别记录在JSON对象中，格式如下：
```json
{
    "request": {
        "method": "GET",
        "uri": "/get",
        "url": "http://httpbin.org:8000/get",
        "size": "75",
        "querystring": {},
        "headers": {
            "accept": "*/*",
            "host": "httpbin.org",
            "user-agent": "curl/7.37.1"
        },
        "tls": {
            "version": "TLSv1.2",
            "cipher": "ECDHE-RSA-AES256-GCM-SHA384",
            "supported_client_ciphers": "ECDHE-RSA-AES256-GCM-SHA384",
            "client_verify": "NONE"
        }
    },
    "upstream_uri": "/",
    "response": {
        "status": 200,
        "size": "434",
        "headers": {
            "Content-Length": "197",
            "via": "kong/0.3.0",
            "Connection": "close",
            "access-control-allow-credentials": "true",
            "Content-Type": "application/json",
            "server": "nginx",
            "access-control-allow-origin": "*"
        }
    },
    "tries": [
        {
            "state": "next",
            "code": 502,
            "ip": "127.0.0.1",
            "port": 8000
        },
        {
            "ip": "127.0.0.1",
            "port": 8000
        }
    ],
    "authenticated_entity": {
        "consumer_id": "80f74eef-31b8-45d5-c525-ae532297ea8e",
        "id": "eaa330c0-4cff-47f5-c79e-b2e4f355207e"
    },
    "route": {
        "created_at": 1521555129,
        "hosts": null,
        "id": "75818c5f-202d-4b82-a553-6a46e7c9a19e",
        "methods": null,
        "paths": [
            "/example-path"
        ],
        "preserve_host": false,
        "protocols": [
            "http",
            "https"
        ],
        "regex_priority": 0,
        "service": {
            "id": "0590139e-7481-466c-bcdf-929adcaaf804"
        },
        "strip_path": true,
        "updated_at": 1521555129
    },
    "service": {
        "connect_timeout": 60000,
        "created_at": 1521554518,
        "host": "example.com",
        "id": "0590139e-7481-466c-bcdf-929adcaaf804",
        "name": "myservice",
        "path": "/",
        "port": 80,
        "protocol": "http",
        "read_timeout": 60000,
        "retries": 5,
        "updated_at": 1521554518,
        "write_timeout": 60000
    },
    "workspaces": [
        {
            "id":"b7cac81a-05dc-41f5-b6dc-b87e29b6c3a3",
            "name": "default"
        }
    ],
    "consumer": {
        "username": "demo",
        "created_at": 1491847011000,
        "id": "35b03bfc-7a5b-4a23-a594-aa350c585fa8"
    },
    "latencies": {
        "proxy": 1430,
        "kong": 9,
        "request": 1921
    },
    "client_ip": "127.0.0.1",
    "started_at": 1433209822425
}
```

关于上述JSON对象的一些注意事项：

- `request` 包含有关客户端发送的请求的属性
- `response` 包含有关发送给客户端的响应的属性
- `tries` 包含负载均衡器对此请求进行的（重试）（成功和失败）列表
- `route` 包含有关请求的特定 Route 的Kong属性
- `service` 包含与所请求 Route 相关的 Service 的Kong属性
- `authenticated_entity` 包含有关已认证凭据的Kong属性（如果已启用身份验证插件
- `workspaces`包含与所请求  Route 关联的工作区的Kong属性。**仅限于Kong Enterprise版本> = 0.34。**
- `consumer` 包含经过身份验证的 Consumer（如果已启用身份验证插件）
- `latencies` 包含一些有关延迟的数据
	- `proxy` 是最终服务处理请求所花费的时间
	- `kong` 是运行所有插件所需的内部Kong延迟
	- `request` 是从客户端读取第一个字节到将最后一个字节发送到客户端之间经过的时间。对于检测请求慢的客户端非常有用。
- `client_ip` 包含原始客户端IP地址
- `started_at` 包含开始处理请求的时间的UTC时间戳。


## Kong Process Errors

此日志记录插件将仅记录HTTP请求和响应数据。
如果要查找Kong进程错误文件（即nginx错误文件），则可以在以下路径中找到它：`$KONG_PREFIX/logs/error.log`，其中`$ KONG_PREFIX`表示[配置的前缀](https://docs.konghq.com/1.3.x/configuration/#prefix)。















