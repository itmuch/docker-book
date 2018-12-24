# Label和Selector（Label和选择器）

*Label*是附加到对象（如Pod）的键值对。Label旨在用于指定对用户有意义的对象的识别属性，但不直接表示核心系统的语义。Label可用于组织和选择对象的子集。Label可在创建时附加到对象，也可在创建后随时添加和修改。每个对象都可定义一组Label。 对于给定的对象，Key必须唯一。

```json
"labels" :   { 
   "key1"   :   "value1" , 
   "key2"   :   "value2" 
 } 
```

我们最终会对Label进行索引和反向索引，以便于高效的查询、watch、排序、分组等操作。不要使用非标识的、大型的结构化数据污染Label。对于非标识的信息应使用非标识，特别是大型和/或结构化数据来污染Label。 非识别信息应使用 [annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 记录。





## 动机

Label使用户能够以松耦合的方式，将自己的组织结构映射到系统对象上，客户端无需存储这些映射。

服务部署和批处理流水线通常是多维实体（例如：多个分区或部署、多个发布轨道、多个层、每层有多个微服务）。管理往往需要跨部门才能进行，这打破了严格层级表现的封装，特别是由基础设施而非用户确定的刚性层次结构。

示例Label：

- `"release" : "stable"`, `"release" : "canary"` 
- `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
- `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
- `"partition" : "customerA"`, `"partition" : "customerB"`
- `"track" : "daily"`, `"track" : "weekly"`

以上只是常用Label示例。可自由定制。请记住，给定对象的Label的key必须唯一。





## 语法和字符集

*Label*是键值对的形式。

有效Label的key分为两段：

* 名称段和前缀段，以 `/` 分隔。
* 名称段是必需的，不超过63个字符，以字母或数字（ `[a-z0-9A-Z]` ）开头和结尾，中间可包含 `-` 、 `_` 、 `.` 、字母或数字等字符。
* 前缀段是可选的。必须是DNS子域：一系列由 `.` 分隔的DNS Label，总共不超过253个字符，后跟 `/` 。 
* 如省略前缀，则假定Label的Key对用户是私有的。
* 为最终用户对象添加Label的自动化系统组件（例如， `kube-scheduler` 、 `kube-controller-manager` 、`kube-apiserver` 、 `kubectl` 或其他第三方自动化组件）必须指定前缀。 `kubernetes.io/` 前缀保留给Kubernetes核心组件。


有效的Label值必须：

* 不超过63个字符
* 为空，或者：以字母或数字（ `[a-z0-9A-Z]` ）开头和结尾，中间包含 `-` 、 `_` 、 `.` 、字母或数字等字符。






## Label选择器

不同于  [names and UIDs](https://kubernetes.io/docs/user-guide/identifiers) ，Label不提供唯一性。一般来说，多个可对象携带相同的Label。

通过*Label选择器* ，客户端/用户可识别一组对象。Label选择器是Kubernetes中的核心分组API。

API目前支持两种类型的选择器：*equality-based* 和 *set-based* 。Label选择器可由逗号分隔的多个*需求*组成。在多重需求的情况下，必须满足所有需求，因此逗号作为*AND*逻辑运算符。

一个empty Label选择器（即一个零需求的选择器）选择集合中的每个对象。

null Label选择器（仅用于可选的选择器字段）不选择任何对象。

**注意** ：两个Controller的Label选择器不能在命名空间内重叠，否则它们将会产生冲突。



### *Equality-based* requirement

*Equality-* or *inequality-based* requirement允许通过LabelKey和Value进行过滤。匹配对象必须满足所有的Label约束，尽管对象可能还有其他Label。允许使用三种运算符：`=` 、 `==` 、 `!=` 。前两个表示*相等* （只是同义），而后者则表示*不相等* 。 例如：

```properties
environment = production
tier != frontend
```

前者选择所有与key = `environment` 并且value = `production` 的资源。后者选择key = `tier` 并且value不等于`frontend` ，以及key不等于`tier` 的所有资源。可使用`,` 过滤是生产环境中（`production` ）非前端（`frontend` ）的资源：`environment=production,tier!=frontend` 。



### *Set-based* requirement

*Set-based* label requirement允许根据一组Value过滤Key。支持三种运算符： `in` 、`notin` 和`exists`（只有Key标识符）。 例如：

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

第一个示例选择Key等于`environment` ，Value等于`production` 或`qa` 的所有资源。

第二个示例选择Key等于`tier` ，Value不等于`frontend` 或`backend`，以及所有Key不等于`tier` 的所有资源。

第三个示例选择Key包含`partition` 所有资源；不检查Value的值。

第四个示例选择Key不包含`partition` 的所有资源，不检查任何值。 

类似地，逗号可用作*AND*运算符。 因此可用`partition,environment notin (qa)` 过滤 Key = `partition`（无论值）并且`environment` != `qa` 的资源。 基于*集合的*Label选择器，也可表示基于等式的Label选择。例如，`environment=production`等同于`environment in (production)` ; 同样，`!=` 等同于`notin` 。

*Set-based* requirement可以与*equality-based* requirement混合使用。例如： `partition in (customerA, customerB),environment!=qa` 。





## API

### LIST与WATCH过滤

LIST和WATCH操作可指定Label选择器，这样就可以使用查询参数过滤返回的对象集。两种requirement都是允许的（在这里表示，它们将显示在URL查询字符串中）：

- *equality-based* requirement： `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
- *set-based* requirements： `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`

两种Label选择器都可用于在REST客户端中LIST或WATCH资源。例如，使用`kubectl` 定位`apiserver` 并使用 *equality-based* 的方式可写为：

```
$ kubectl get pods -l environment=production,tier=frontend
```

或使用*set-based* requirements：

```
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
```

如上所示，*set-based* requirement表达能力更强。例如，他们可以对Value执行*OR*运算符：

```
$ kubectl get pods -l 'environment in (production, qa)'

```

或通过exists操作符，实现restricting negative matching：

```
$ kubectl get pods -l 'environment,environment notin (frontend)'
```



### 在API对象中设置引用

一些Kubernetes对象（如  [`services`](https://kubernetes.io/docs/user-guide/services) 和 [`replicationcontrollers`](https://kubernetes.io/docs/user-guide/replication-controller) ）也可使用Label选择器来指定其他资源（例如 [pods](https://kubernetes.io/docs/user-guide/pods) ）。





#### Service 和 ReplicationController

使用Label选择器定义`service` 指向的一组Pod。类似地， `replicationcontroller` 所管理的Pod总数也可用Label选择器定义。

两个对象的Label选择器都使用map在`json` 或`yaml` 文件中定义，只支持*equality-based* requirement选择器：

```
"selector": {
    "component" : "redis",
}

```

或：

```
selector:
    component: redis
```

此选择器（分别以`json` 或`yaml` 格式）等价于 `component=redis` 或`component in (redis)` 。



#### 支持set-based requirement的资源

较新的资源（例如 [`Job`](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) 、 [`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) 、 [`Replica Set`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) 以及 [`Daemon Set`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) ）也支持 *set-based* requirement。

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

`matchLabels` 是`{ key,value }` 的映射。 `matchLabels` 映射中的单个`{ key,value }` 等价于`matchExpressions`一个元素，其`key` 为“key”， `operator` 为“In”， `values` 数组仅包含“value”。 `matchExpressions` 是一个Pod选择器需求列表。有效运算符包括In、NotIn、Exists以及DoesNotExist。 当使用In和NotIn时，Value必须非空。所有来自`matchLabels` 和`matchExpressions` 的需求使用AND串联，所有需求都满足才能匹配。



#### 选择Node列表

使用Label选择Node的一个用例是约束一个Pod能被调度到哪些Node上。有关详细信息，请参阅有关 [node selection](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 的文档。