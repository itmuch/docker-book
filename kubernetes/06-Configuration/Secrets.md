# Secrets

`secret` 类型的对象旨在保存敏感信息，例如密码、OAuth token和ssh key。将这些信息放在 `secret` 中比放在 `pod` 定义或Docker镜像中更加安全、灵活。有关更多信息，请参阅 [Secrets design document](https://git.k8s.io/community/contributors/design-proposals/auth/secrets.md) 。





## Overview of Secrets（Secret概述）

Secret是包含少量敏感数据（如密码、token或key）的对象。这样的信息可能会被放在Pod spec或镜像中；将其放在一个Secret对象中能够更好地控制如何使用它，并降低意外暴露的风险。

用户可以创建Secret，系统也会创建一些Secret。

要使用Secret，Pod需要引用Secret。Secret与Pod一起使用有两种方式：作为mount到一个或多个容器上的  [volume](https://kubernetes.io/docs/concepts/storage/volumes/) 的文件，或在为Pod拉取镜像时由kubelet使用。



### Built-in Secrets（内置的Secret）

#### Service Accounts Automatically Create and Attach Secrets with API Credentials（服务帐户自动使用API Credential创建和附加Secret）

Kubernetes自动创建包含访问API的凭据的Secret，并自动修改您的Pod，让其使用此类型的Secret。

如果需要，自动创建和API Credential的使用可禁用或覆盖。但是，如果你只是想安全访问apiserver，默认方式是推荐的工作流程。

参阅 [Service Account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) 文档查看其如何工作的更多信息。





### Creating your own Secrets（创建自己的Secret）

#### Creating a Secret Using kubectl create secret（使用kubectl create secret命令创建Secret）

假设Pod需要访问数据库。Pod所使用的用户名和密码在本地机器上的 `./username.txt` 和 `./password.txt` 文件中。

```shell
# Create files needed for rest of example.
$ echo -n "admin" > ./username.txt
$ echo -n "1f2d1e2e67df" > ./password.txt
```

`kubectl create secret` 命令可将这些文件打包成一个Secret，并在Apiserver上创建对象。

```shell
$ kubectl create secret generic db-user-pass --from-file=./username.txt --from-file=./password.txt
secret "db-user-pass" created
```

你可以检查这个Secret是如何创建的：

```shell
$ kubectl get secrets
NAME                  TYPE                                  DATA      AGE
db-user-pass          Opaque                                2         51s

$ kubectl describe secrets/db-user-pass
Name:            db-user-pass
Namespace:       default
Labels:          <none>
Annotations:     <none>

Type:            Opaque

Data
====
password.txt:    12 bytes
username.txt:    5 bytes
```

请注意，默认情况下， `get` 和 `describe` 都不会显示文件的内容。这是为了保护Secret不被意外地暴露。

请参阅 [decoding a secret](https://kubernetes.io/docs/concepts/configuration/secret/#decoding-a-secret) ，了解如何查看内容。

#### Creating a Secret Manually（手动创建Secret）

您也可以先以json或yaml格式在文件中创建一个Secret对象，然后再创建该对象。

每一项都必须以base64进行编码：

```shell
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```

现在写一个如下所示的Secret对象：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

data字段是一个map。它的key必须符合 [`DNS_SUBDOMAIN`](https://git.k8s.io/community/contributors/design-proposals/architecture/identifiers.md) ，只是也允许leading dots。value是任意数据，使用base64进行编码。

使用 [`kubectl create`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#create) 创建Secret：

```shell
$ kubectl create -f ./secret.yaml
secret "mysecret" created
```

**Encoding Note：**Secret数据的值在序列化成JSON或YAML后，被编码为base64字符串。换行符在这些字符串中无效，必须省略。当在Darwin/OS X上使用 `base64` utility时，用户应避免使用 `-b` 选项来拆分非常长的行。相反，Linux用户*应该*添加选项 `-w 0` 到 `base64` 命令；或者管道 `base64 | tr -d '\n'` ，如果 `-w` 选项不可用。

#### Decoding a Secret（解码Secret）

Secret可通过 `kubectl get secret` 命令查看。例如，要查看在上一节中创建的Secret：

```yaml
$ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
kind: Secret
metadata:
  creationTimestamp: 2016-01-22T18:41:56Z
  name: mysecret
  namespace: default
  resourceVersion: "164619"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: cfee02d6-c137-11e5-8d73-42010af00002
type: Opaque
```

解码password字段：

```shell
$ echo "MWYyZDFlMmU2N2Rm" | base64 --decode
1f2d1e2e67df
```



### Using Secrets（使用Secret）

Secret可作为数据volume被mount，或作为环境变量暴露，以供Pod中的容器使用。 它们也可被系统的其他部分使用，而不会直接暴露在Pod内。例如，它们可保存系统其他部分应该使用的凭据，从而代表你外部系统进行交互。





#### Using Secrets as Files from a Pod（使用Secret作为来自Pod的文件）

在Pod中的volume中使用Secret：

1. 创建一个secret或使用现有的secret。多个Pod可引用相同的secret。
2. 修改您的Pod定义，在 `spec.volumes[]` 下添加一个Volume。为Volume任意起个名字，并设置一个 `spec.volumes[].secret.secretName` 字段等于Secret对象的名称。
3. 将 `spec.containers[].volumeMounts[]` 添加到需要使用Secret的每个容器。 设置`spec.containers[].volumeMounts[].readOnly = true` ，设置 `spec.containers[].volumeMounts[].mountPath` 为您希望显示Secret的未使用的目录名称。
4. 修改您的镜像和/或命令行，以便程序查找该目录中的文件。 Secret `data` map中的每个key都将成为 `mountPath` 下的文件名。

以下是一个在Volume挂载Secret的Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
```

你想使用的每个Secret都需要在 `spec.volumes` 中引用。

如果Pod中有多个容器，则每个容器都需要自己的 `volumeMounts` ，但每个Secret只需要一个 `spec.volumes` 。

您可以将多个文件打包成一个Secret，或使用多个Secret，怎么方便怎么玩。

**将Secret key投影到特定路径**

我们还可控制投影Secret key的Volume。可使用 `spec.volumes[].secret.items` 字段来更改每个key的目标路径：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
```

这样：

- `username` 这个Secret将会存储在 `/etc/foo/my-group/my-username` 文件中，而非 `/etc/foo/username` 。
- `password` 不被投影

如使用 `spec.volumes[].secret.items` ，则只会投影 `items` 中指定的key。如果要投影Secret中的所有key，那么所有key都必须列在 `items` 字段中。 所有列出的key必须存在于相应的Secret中。否则，Volume不会被创建。

**Secret files permissions** （**Secret文件权限**）

你也可以指定一个Secret的权限模式位。如不指定，默认使用 `0644` 。可指定整个Secret Volume的默认模式，并根据需要覆盖每个key。

例如，你可以指定一个像这样的默认模式：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      defaultMode: 256
```







// TODO

Then, the secret will be mounted on `/etc/foo` and all the files created by the secret volume mount will have permission `0400`.

Note that the JSON spec doesn’t support octal notation, so use the value 256 for 0400 permissions. If you use yaml instead of json for the pod, you can use octal notation to specify permissions in a more natural way.

You can also use mapping, as in the previous example, and specify different permission for different files like this:








## 原文

<https://kubernetes.io/docs/concepts/configuration/secret/>