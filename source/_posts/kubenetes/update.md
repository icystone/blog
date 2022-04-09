---
title: kubenetes 资源更新
copyright: true
date: 2022-04-09 15:20:00
tags:
categories:
- kubenetes
---

# k8s 更新机制
 `metadata.resourceVersion`：每个资源从创建开始都有一个 resourceVersion 字段，代表该资源在下层数据库中存储的版本，每次修改，版本号都会发生变化。   
> [资源版本](https://kubernetes.io/zh/docs/reference/using-api/api-concepts/#resource-versions)中指出，该版本号是一个k8s的内部机制，用户不能假定资源版本是某种数值标识，不可以对两个资源版本值进行比较大小。

k8s 要求用户 update 的对象必须带有 resourceVersion。并且在 update 时会比较`提交的 resourceVersion` 与`当前 k8s 中这个对象最新的 resourceVersion`一致，才能接收本次 update。否则 api server 会拒绝请求并告诉用户发生了 confilct。

# k8s apply
## client-side apply
首先解析用户提交的数据(YAML/JSON)为一个对象A；然后调用 Get 接口从 k8s 中查询这个资源对象；
* 如果查询结果不存在，kubectl 将本次用户提交的数据记录到`对象 A 的 annotation 中`(key 为 `kubectl.kubernetes.io/last-applied-configuration`)，这些内容用来计算配置文件中已经移除的字段。最后将对象 A 提交给 k8s 创建；
* 如果查询到 k8s 中已有这个资源，假设为对象 B：
    1. kubectl 尝试从对象 B 的 annotation 中取出 `kubectl.kubernetes.io/last-applied-configuration`（对应了上一次提交的内容）
    1. kubectl 根据前一次 apply 的内容和本次 apply 的内容计算出要删除的字段和发生过修改的字段，得到 patch Request
    1. 将新的配置文件添加到本次的 `kubectl.kubernetes.io/last-applied-configuration`，最后用 patch 请求提交给 k8s(默认为 strategic，如果是 CRD 使用 merge)

## server-side apply
从 Kubernetes v1.18 开始取代 client-side apply。   
添加 `metadata.managedFields` 来明确指定谁管理哪些资源字段。当使用 `server-side apply`时，尝试去改变一个被其他人管理的字段，会导致请求被拒绝。   
如果两个或以上的应用者均把同一个字段设置为相同值，他们将共享此字段的所有权。 后续任何改变共享字段值的尝试，不管由那个应用者发起，都会导致`冲突`。 共享字段的所有者可以放弃字段的所有权，这只需从配置文件中删除该字段即可。   
`冲突`是一种特定的错误状态， 发生在执行 Apply 改变一个字段，而恰巧该字段被其他用户声明过主权时。 这可以防止一个应用者不小心覆盖掉其他用户设置的值。 冲突发生时，应用者有三种办法来解决它：

* 覆盖前值，成为唯一的管理器： 如果打算覆盖该值（或应用者是一个自动化部件，比如控制器）， 应用者应该设置查询参数 force 为 true，然后再发送一次请求。 这将强制操作成功，改变字段的值，从所有其他管理器的 managedFields 条目中删除指定字段。

* 不覆盖前值，放弃管理权： 如果应用者不再关注该字段的值， 可以从配置文件中删掉它，再重新发送请求。 这就保持了原值不变，并从 managedFields 的应用者条目中删除该字段。

* 不覆盖前值，成为共享的管理器： 如果应用者仍然关注字段值，并不想覆盖它， 他们可以在配置文件中把字段的值改为和服务器对象一样，再重新发送请求。 这样在不改变字段值的前提下， 就实现了字段管理被应用者和所有声明了管理权的其他的字段管理器共享。

更多参考[server-side-apply](https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/)

# k8s patch
当用户对某个资源对象提交一个 patch 请求时，api server 不会考虑版本问题，直接接收 patch 请求，并更新版本号。   
目前 k8s 提供了 3 种 patch 策略：json、merge、strategic
## json
JSON patch是执行在资源对象上的一系列操作，如下所示：
```json
[
  { "op": "replace", "path": "/baz", "value": "boo" },
  { "op": "add", "path": "/hello", "value": ["world"] },
  { "op": "remove", "path": "/foo" }
]
```
细节参考：[JSON patch](http://jsonpatch.com/)   
Notice：如果其中任何一个操作失败，那么整个 patch 都不成功。
## merge
merge patch包含一个对资源对象的`部分 json 描述`，该 json 对象被提交到服务端，并和服务端的对象合并。

原始JSON：
```json
 {
         "title": "Goodbye!",
         "author" : {
       "givenName" : "John",
       "familyName" : "Doe"
         },
         "tags":[ "example", "sample" ],
         "content": "This will be unchanged"
       }
```
Patch Request:
```json
{
         "title": "Hello!",
         "phoneNumber": "+01-123-456-7890",
         "author": {
       "familyName": null
         },
         "tags": [ "example" ]
       }
```
Result:
```json
{
         "title": "Hello!",
         "author" : {
       "givenName" : "John"
         },
         "tags": [ "example" ],
         "content": "This will be unchanged",
         "phoneNumber": "+01-123-456-7890"
       }
```
细节参考：[merge patch](https://datatracker.ietf.org/doc/html/rfc7386)   
Notice：如果 merge 的目标不是 object，那么执行的操作是 replace，比如数组。
## strategic
k8s 独有的策略，避免了 merge Patch 直接被 replace 的行为。   
原始：
```yaml
spec:
  template:
    spec:
      containers:
      - name: patch-demo-ctr
        image: nginx
```
Patch Request：
```yaml
spec:
  template:
    spec:
      containers:
      - name: patch-demo-ctr-2
        image: redis
```
Result:
```yaml
spec:
  template:
    spec:
      containers:
      - name: patch-demo-ctr
        image: nginx
      - name: patch-demo-ctr-2
        image: redis
```

在使用 strategic Patch 时，list 到底 replace or merge 取决于它的 `patchStrategy`   
patchStrategy 目前支持两种：   
1. merge 在 patch 数组时可以根据指定的 name 来 patch，避免整个数组 replace
2. retainKeys 必须显式地声明替换某个key

For example:
```golang
type PodSpec struct {
  ...
  Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" ...`
```
Notice：CRD 不支持 strategic Patch。   

[使用参考](https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)   
可以通过[官方API文档](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/#podspec-v1-core)查看哪些字段支持 merge、retainKeys   
