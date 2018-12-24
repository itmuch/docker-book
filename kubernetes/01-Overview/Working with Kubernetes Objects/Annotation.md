# Annotation（注解）

可使用Kubernetes annotation将任意的非识别元数据附加到对象。 客户端可取得此元数据。





## 将元数据附加到对象

您可以使用标签或Annotation将元数据附加到Kubernetes对象。标签可用于选择对象并查找满足某些条件的对象集合。相比之下，Annotation不用于标识和选择对象。Annotation中的元数据可大可小，结构化或非结构化，并且可包括标签所不允许的字符。

Annotation类似于标签，是键值对映射：

```json
"annotations": {
  "key1" : "value1",
  "key2" : "value2"
}
```

类似以下信息可记录到Annotation中：

- 由declarative configuration layer管理的字段。将这些字段附加为Annotation，可将它们与客户端或服务器设置的默认值、自动生成的字段或以及auto-sizing或auto-scaling的系统所设置的字段区分开。
- 构建信息、发布信息或镜像信息，如时间戳、release ID、git分支、PR编号、镜像哈希以及注册表地址。
- 指向日志、监控、分析或审计仓库。


- 用于调试的客户端库或工具的信息：例如名称、版本和构建信息。
- 用户或工具/系统来源信息，例如来自其他生态系统组件的相关对象的URL。
- 轻量级升级工具的元数据：例如配置或检查点。
- 负责人的电话或寻呼机号码，或指定能找到该信息的条目，例如团队网站。

如果不使用注释，则可将这种类型的信息存储在外部数据库或目录中，但这将会使生产共享客户端库/工具（用于部署、管理、内省）变得更加困难。



## 原文

<https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/>