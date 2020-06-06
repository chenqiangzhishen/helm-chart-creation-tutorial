Table of Contents
=================

* [Helm Chart 创作指南](#helm-chart-创作指南)
* [Helm V2 与 V3 主要区别](#helm-v2-与-v3-主要区别)
    * [1、架构性变化 - 去除了 Tiller](#1架构性变化---去除了-tiller)
    * [2、Tiller 变更引入的新变化 - Release 不再是全局资源](#2tiller-变更引入的新变化---release-不再是全局资源)
    * [3、Values 支持 JSON Schema 校验器](#3values-支持-json-schema-校验器)
* [环境准备](#环境准备)
* [开始创作](#开始创作)
* [走近Chart](#走近chart)
* [打包使用](#打包使用)
* [进阶使用](#进阶使用)
* [其他](#其他)


### Helm Chart 创作指南

Helm 作为 Kubernetes 体系的包管理工具，已经逐渐成为了事实上的应用分发标准。根据 2018 年 CNCF 的一项云原生用户调研，超过百分之六十八用户选择 Helm 来作为应用打包交付方式。在开源社区中，越来越多的软件被搬迁到 Kubernetes 集群上，它们中的绝大部分都是通过 Helm 来进行交付的。
Helm作为当前最流行的Kubernetes应用管理工具之一，整合应用部署所需的K8s资源（包括Deployment，Service等）到Chart中。今天，本文会带领大家学习如何创建一个简单的Chart。

---

#### Helm V2 与 V3 主要区别

##### 1、架构性变化 - 去除了 Tiller

在 Helm 2 中，Tiller 是作为一个 Deployment 部署在 kube-system 命名空间中，很多情况下，我们会为 Tiller 准备一个 ServiceAccount ，这个 ServiceAccount 通常拥有集群的所有权限。用户可以使用本地 Helm 命令，自由地连接到 Tiller 中并通过 Tiller 创建、修改、删除任意命名空间下的任意资源。

然而在多租户场景下，这种方式也会带来一些安全风险，我们即要对这个 ServiceAccount 做很多剪裁，又要单独控制每个租户的控制，这在当前的 Tiller 模式下看起来有些不太可能。

于是在 Helm 3 中，Tiller 被移除了。新的 Helm 客户端会像 kubectl 命令一样，读取本地的 kubeconfig 文件，使用我们在 kubeconfig 中预先定义好的权限来进行一系列操作。这样做法即简单，又安全。

虽然 Tiller 文件被移除了，但 Release 的信息仍在集群中以 ConfigMap 的方式存储，因此体验和 Helm 2 没有区别。

##### 2、Tiller 变更引入的新变化 - Release 不再是全局资源

在 Helm 2 中，Tiller 自身部署往往在 kube-system 下，虽然不一定是 cluster-admin 的全局管理员权限，但是一般都会有 kube-system 下的权限。当 Tiller 想要存储一些信息的时候，它被设计成在 kube-system 下读写 ConfigMap 。

在 Helm 3 中，Helm 客户端使用 kubeconfig 作为认证信息直接连接到 Kubernetes APIServer，不一定拥有 cluster-admin 权限或者写 kube-system 的权限，因此它只能将需要存储的信息存在当前所操作的 Kubernetes Namespace 中，继而 Release 变成了一种命名空间内的资源。

##### 3、Values 支持 JSON Schema 校验器

Helm Charts 是一堆 Go Template 文件、一个变量文件 Values 和一些 Charts 描述文件的组合。Go Template 和 Kubernetes 资源描述文件的内容十分灵活，在开发迭代过程中，很容易出现一些变量未定义的问题。Helm 3 引入了 JSON Schema 校验，它支持用一长串 DSL 来描述一个变量文件的格式、检查所有输入的变量的格式。

当我们运行 `helm install` 、 `helm upgrade` 、 `helm lint` 、 `helm template` 命令时，JSON Schema 的校验会自动运行，如果失败就会立即报错。

---

#### 环境准备

本文基于的是最新的[Helm v3.2.1](https://v3.helm.sh/)，Helm v3相较于Helm v2有较大的变动，比如Helm v3没有服务器端的Tiller组件，在Chart的使用上接口也有一些变化。建议没有Helm v3的同学先从Helm的Github仓库中下载最新的[Helm v3 release](https://github.com/helm/helm/releases/tag/v3.2.1)。想要仔细钻研Helm v3各种特性的同学，可以参考Helm v3的完整[官方文档](https://v3.helm.sh/docs/using_helm/)。

在下载得到最新的Helm v3后，我们再也不用像 V2 版本那样需要先运行`helm init`来初始化Helm进行安装 Tiller。
而V3初始化会自动在我们使用的时候完成。

---

#### 开始创作

首先，我们需要有一个要部署的应用。这里我们使用一个简单的基于golang的[hello world HTTP服务](https://github.com/chenqiangzhishen/helm-chart-creation-tutorial/blob/master/src/main.go)。该服务通过读取环境变量`USERNAME`获得用户自己定义的名称，然后监听80端口。对于任意HTTP请求，返回`Hello ${USERNAME}。`比如如果设置`USERNAME=world`（默认场景），该服务会返回`Hello world`。

准备好要部署的应用镜像后，运行`helm create my-hello-world`，便会得到一个helm自动生成的空chart。这个chart里的名称是`my-hello-world`。
**需要注意的是，Chart里面的my-hello-world名称需要和生成的Chart文件夹名称一致。如果修改my-hello-world，则需要做一致的修改。**
现在，我们看到Chart的文件夹目录如下

```yaml
my-hello-world/
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

在根目录下的Chart.yaml文件内，声明了当前Chart的名称、版本等基本信息，这些信息会在该Chart被放入仓库后，供用户浏览检索。比如我们可以把Chart的Description改成"My first hello world helm chart"。

---

#### 走近Chart

Helm Chart对于应用的打包，不仅仅是将Deployment和Service以及其它资源整合在一起。我们看到deployment.yaml和service.yaml文件被放在templates/文件夹下，相较于原生的Kubernetes配置，多了很多渲染所用的可注入字段。比如在deployment.yaml的`spec.replicas`中，使用的是`.Values.replicaCount`而不是Kubernetes本身的静态数值。这个用来控制应用在Kubernetes上应该有多少运行副本的字段，在不同的应用部署环境下可以有不同的数值，而这个数值便是由注入的`Values`提供。

在根目录下我们看到有一个`values.yaml`文件，这个文件提供了应用在安装时的默认参数。在默认的`Values`中，我们看到`replicaCount: 1`说明该应用在默认部署的状态下只有一个副本。

为了使用我们要部署应用的镜像，我们看到deployment.yaml里在`spec.template.spec.containers`里，`image`和`imagePullPolicy`都使用了`Values`中的值。其中`image`字段由`.Values.image.repository`和`.Chart.AppVersion`组成。看到这里，应该就知道我们需要变更的字段了，一个是位于values.yaml内的`image.repository`，另一个是位于Chart.yaml里的`AppVersion`。我们将它们与我们需要部署应用的docker镜像匹配起来。这里我们把values.yaml里的`image.repository`设置成`somefive/hello-world`，把Chart.yaml里的`AppVersion`设置成`1.0.0`即可。

类似的，我们可以查看service.yaml内我们要部署的服务，其中的主要配置也在values.yaml中。默认生成的服务将80端口暴露在Kubernetes集群内部。我们暂时不需要对这一部分进行修改。

由于部署的hello-world服务会从环境变量中读取`USERNAME`环境变量，我们将这个配置加入deployment.yaml。相关部分如下：

```yaml
- name: {{ .Chart.Name }}
  image: "{{ .Values.image.repository }}:{{ .Chart.AppVersion }}"
  imagePullPolicy: {{ .Values.image.pullPolicy }}
  env:
    - name: USERNAME
      value: {{ .Values.Username }}
```

现在我们的deployment.yaml模版会从values.yaml中加载`Username`字段，因此相应的，我们也在values.yaml中添加`Username: YourName`。

---

#### 打包使用

完成上述配置后，我们可以使用`helm lint --strict my-hello-world`进行格式检查。如果显示

```bash
==> Linting my-hello-world/
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

那么我们就已经离成功只差一步之遥了。

接下来，我们运行`helm package my-hello-world`指令对我们的Chart文件夹进行打包。现在我们就得到了`my-hello-world-0.1.0.tgz`的Chart包。到这一步我们的Chart便已经完成了。

等等，我们忘记制作前面提到的镜像了，现在需要从[hello world HTTP服务](https://github.com/chenqiangzhishen/helm-chart-creation-tutorial/blob/master/src/main.go)
下载到本地，然后进行构建：

```bash
# 这里以我的 dockerhub 中的 repo `qzschen` 为例，tag 1.0.0 是上面给定的 apiVersion 版本号
docker build . -t qzschen/hello-world:1.0.0
# 提前准备好你在 dockerhub 或者 私有镜像仓库的账号，可能需要先登录一下
docker login -u qzschen
docker push qzschen/hello-world:1.0.0
```

现在我们来尝试安装下这个 Chart:
运行`helm install my-hello-world-chart-test my-hello-world-0.1.0.tgz`来将本地的chart安装到my-hello-world-chart-test的Release中。运行`kubectl get pods`我们可以看到要部署的pod已经处于运行状态

```bash
NAME                                         READY   STATUS    RESTARTS   AGE
my-hello-world-chart-test-5487c7765d-624z8  1/1     Running   0          4m3s
```

然后我们根据提示执行下面的命令：

```bash
$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=my-hello-world,app.kubernetes.io/instance=my-hello-world-chart-test2" -o jsonpath="{.items[0].metadata.name}")
$ kubectl port-forward $POD_NAME 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

就可以直接在本地运行`curl localhost:8080`看到`Hello YourName`了！

```bash
# 打开另一个控制台
$ curl localhost:8080
Hello YourName
```

---

#### 进阶使用

上述提到values.yaml只是Helm install参数的默认设置，我们可以在安装Chart的过程中使用自己的参数覆盖。比如我们可以运行`helm install my-hello-world-chart-test2 my-hello-world-0.1.0.tgz --set Username="Cloud Native"`来安装一个新Chart。同样运行`kubectl port-forward`进行端口映射，这时可以得到`Hello Cloud Native`。

我们注意到在安装Chart指令运行后，屏幕的输出会出现

```bash
NAME: my-hello-world-chart-test2
LAST DEPLOYED: Sat Jun  6 19:56:03 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=my-hello-world,app.kubernetes.io/instance=my-hello-world-chart-test2" -o jsonpath="{.items[0].metadata.name}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:80
```

#### 其他

Helm Chart还有诸如dependency等其他功能，更加详细的资料可以参考Helm官方文档的[相关章节](https://helm.sh/docs/chart_template_guide/)。