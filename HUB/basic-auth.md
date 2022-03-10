# 基础认证

> 本文原文链接：https://docs.konghq.com/hub/kong-inc/basic-auth/

使用用户名和密码保护将基础认证添加到 Service 或 Route 。该插件将检查`Proxy-Authorization`和`Authorization` header 中的有效凭据（按此顺序）。

> 注意：此插件的功能与0.14.1之前的Kong版本和0.34之前的Kong Enterprise版本捆绑在一起，与此处记录的不同。
有关详细信息，请参阅[CHANGELOG](https://github.com/Kong/kong/blob/master/CHANGELOG.md)。

## 术语

- `plugin`: 在请求被代理到上游API之前或之后，在Kong内部执行操作的插件。
- `Service`: 表示外部 *upstream* API或微服务的Kong实体。
- `Route`: 表示将下游请求映射到上游服务的方法的Kong实体。
- `upstream service`: 这是指位于Kong后面的您自己的 API/service，转发客户端请求。

## 配置

此插件与具有以下协议的请求兼容：

- `http`
- `https`

此插件与无DB模式**部分兼容**。

可以使用声明性配置创建使用者和凭据。
在凭据上执行POST，PUT，PATCH或DELETE的Admin API端点在无DB模式下不可用。


## 在 Service 上启用插件

**使用数据库：**

通过发出以下请求在Service上配置此插件：
```bash
$ curl -X POST http://kong:8001/services/{service}/plugins \
    --data "name=basic-auth"  \
    --data "config.hide_credentials=true"
```

**不使用数据库：**

通过添加此部分在服务上配置此插件执行声明性配置文件：

```yaml
plugins:
- name: basic-auth
  service: {service}
  config: 
    hide_credentials: true
```
在这两种情况下，`{service}`是此插件配置将定位的Service的`id`或`name`。

## 在 Route 上启用插件

**使用数据库：**

在Route上配置此插件：

```bash
$ curl -X POST http://kong:8001/routes/{route}/plugins \
    --data "name=basic-auth"  \
    --data "config.hide_credentials=true"
```

**不使用数据库：**

通过添加此部分在路由上配置此插件执行声明性配置文件：

```yaml
plugins:
- name: basic-auth
  route: {route}
  config: 
    hide_credentials: true
```

在这两种情况下，`{route}`是此插件配置将定位的Route的`id`或`name`。


## 全局插件

- **使用数据库：** 可以使用`http://kong:8001/plugins/`配置所有插件。
- **不使用数据库：** 可以通过`plugins: `配置所有插件：声明性配置文件中的条目。

与任何 Service ，Route 或 Consumer （或API，如果您使用旧版本的Kong）无关的插件被视为“全局”，并将在每个请求上运行。有关更多信息，请阅读[插件参考](https://docs.konghq.com/latest/admin-api/#add-plugin)和[插件优先级](https://docs.konghq.com/latest/admin-api/#precedence)部分。

### 参数

以下是可在此插件配置中使用的所有参数的列表：


| 参数 | 默认值 | 描述 |
| ---- | ------ | ---- |
| `name` |  |  要使用的插件的名称，在本例中为`basic-auth`  |
| `service_id` |  | 此插件将定位的 Service 的ID。|
| `route_id` |  |  此插件将定位的 Route 的ID。 |
| `enabled` |  `true` | 是否将应用此插件。  |
| `config.hide_credentials` <br> *optional* | `false` | 一个可选的布尔值，告诉插件显示或隐藏来自上游服务的凭据。如果为`true`，插件将在代理之前从请求中剥离凭证（即`Authorization` header）。 |
| `config.anonymous`  <br> *optional*  | | 如果身份验证失败，则用作“匿名”使用者的可选字符串（使用者uuid）值。如果为空（默认），则请求将失败，并且身份验证失败`4xx`。 请注意，此值必须引用Kong内部的Consumer `id`属性，而不是其`custom_id`。| 

一旦应用后，具有有效凭据的任何用户都可以访问该 Service 。
要仅限某些经过身份验证的用户使用，还要添加https://docs.konghq.com/plugins/acl/[ACL插件]()（此处未介绍）并创建白名单或黑名单用户组。


## 使用方法

要使用该插件，首先需要创建一个Consumer来将一个或多个凭据关联到。Consumer表示使用上游服务的开发人员或应用程序。

### 创建一个 Consumer

您需要将凭证与现有的Consumer对象相关联。消费者可以拥有多个凭据。

**使用数据库：**

要创建使用者，您可以执行以下请求：
```
curl -d "username=user123&custom_id=SOME_CUSTOM_ID" http://kong:8001/consumers/
```
**不使用数据库：**

您的声明性配置文件需要有一个或多个使用者。您可以在`consumers:`上创建它们yaml部分：
```
consumers:
- username: user123
  custom_id: SOME_CUSTOM_ID
```

在这两种情况下，参数如下所述：

| 参数 | 描述 |
| ---- | ---- |
| `username` <br> *semi-optional* |  consumer的用户名。必须指定此字段或`custom_id`。|
| `custom_id` <br> *semi-optional* | 用于将使用者映射到另一个数据库的自定义标识符。必须指定此字段或`username`。|

如果您还将[ACL插件](https://docs.konghq.com/plugins/acl/)和白名单与此服务一起使用，则必须将新使用者添加到列入白名单的组。有关详细信息，请参阅[ ACL: Associating Consumers ](https://docs.konghq.com/plugins/acl/#associating-consumers) 。

### 创建一个 Credential

**使用数据库：**

您可以通过发出以下HTTP请求来配置新的用户名/密码凭据：
```bash
$ curl -X POST http://kong:8001/consumers/{consumer}/basic-auth \
    --data "username=Aladdin" \
    --data "password=OpenSesame"
```

**不使用数据库：**

您可以在`basicauth_credentials`yaml条目上的声明性配置文件中添加凭据：
```bash
basicauth_credentials:
- consumer: {consumer}
  username: Aladdin
  password: OpenSesame
```

在这两种情况下，字段/参数的工作方式如下所示：

| 字段/参数 | 描述 |
| --------- | ---- |
| `{consumer}` | 要将凭据关联到的[Consumer](https://docs.konghq.com/latest/admin-api/#consumer-object)实体的`id`或`username`属性。|
| `username` | 要在基本身份验证中使用的用户名 | 
| `password` <br> *optional* | 在基本身份验证中使用的密码 | 

### 使用 Credential

authorization header 必须是base64编码的。例如，如果凭证使用`Aladdin`作为用户名而`OpenSesame`作为密码，则字段的值是`Aladdin：OpenSesame`或`QWxhZGRpbjpPcGVuU2VzYW1l`的base64编码。

然后，授权（或代理授权）标头必须显示为：

```
Authorization: Basic QWxhZGRpbjpPcGVuU2VzYW1l
```

只需使用标题发出请求：

```bash
$ curl http://kong:8000/{path matching a configured Route} \
    -H 'Authorization: Basic QWxhZGRpbjpPcGVuU2VzYW1l'
```

### 上游 headers

当客户端经过身份验证后，插件会在将请求代理到上游服务之前将一些header添加到请求中，以便您可以在代码中标识Consumer：

- `X-Consumer-ID`，Kong Consumer 的ID
- `X-Consumer-Custom-ID`，Consumer 的 `custom_id`（如果设置）
- `X-Consumer-Username`，Consumer 的 `username`（如果设置）
- `X-Credential-Username`，Credential的用户名（仅当消费者不是'匿名'消费者时）
- `X-Anonymous-Consumer`，身份验证失败时将设置为`true`，并设置“匿名”Consumer。

您可以使用此信息来实现其他逻辑。您可以使用`X-Consumer-ID`值来查询Kong Admin API并检索有关Consumer的更多信息。

## 通过基本认证证书进行分页

> 注意：此功能在Kong 0.11.2中引入。

您可以使用以下请求为所有Consumer分配基本身份验证凭据：

```
$ curl -X GET http://kong:8001/basic-auths

{
    "total": 3,
    "data": [
        {
            "created_at": 1511379926000,
            "id": "805520f6-842b-419f-8a12-d1de8a30b29f",
            "password": "37b1af03d3860acf40bd9c681aa3ef3f543e49fe",
            "username": "baz",
            "consumer": { "id": "5e52251c-54b9-4c10-9605-b9b499aedb47" }
        },
        {
            "created_at": 1511379863000,
            "id": "8edfe5c7-3151-4d92-971f-3faa5e6c5d7e",
            "password": "451b06c564a06ce60874d0ea2f542fa8ed26317e",
            "username": "foo",
            "consumer": { "id": "89a41fef-3b40-4bb0-b5af-33da57a7ffcf" }
        },
        {
            "created_at": 1511379877000,
            "id": "f11cb0ea-eacf-4a6b-baea-a0e0b519a990",
            "password": "451b06c564a06ce60874d0ea2f542fa8ed26317e",
            "username": "foobar",
            "consumer": { "id": "89a41fef-3b40-4bb0-b5af-33da57a7ffcf" }
        }
    ]
}
```

您可以使用此其他路径按Consumer筛选列表：

```
$ curl -X GET http://kong:8001/consumers/{username or id}/basic-auths

{
    "total": 1,
    "data": [
        {
            "created_at": 1511379863000,
            "id": "8edfe5c7-3151-4d92-971f-3faa5e6c5d7e",
            "password": "451b06c564a06ce60874d0ea2f542fa8ed26317e",
            "username": "foo",
            "consumer": { "id": "89a41fef-3b40-4bb0-b5af-33da57a7ffcf" }
        }
    ]
}
```

`username or id`:需要列出凭据的consumer的用户名或ID


## 检索与凭据关联的使用者

> 注意：此功能在Kong 0.11.2中引入。

可以使用以下请求检索与basic-auth Credential关联的Consumer ：

```bash
curl -X GET http://kong:8001/basic-auths/{username or id}/consumer

{
   "created_at":1507936639000,
   "username":"foo",
   "id":"c0d92ba9-8306-482a-b60d-0cfdd2f0e880"
}
```

`username or id` : 要获取关联Consumer的basic-auth Credential的`id`或`username`属性。
请注意，此处接受的`username`不是Consumer的`username`属性。





