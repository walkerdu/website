---
title: 使用 AppArmor 限制容器对资源的访问
content_type: tutorial
weight: 10
---
<!--
title: Restrict a Container's Access to Resources with AppArmor
content_type: tutorial
weight: 10
-->

<!-- overview -->

{{< feature-state for_k8s_version="v1.4" state="beta" >}}

<!-- 
AppArmor is a Linux kernel security module that supplements the standard Linux user and group based
permissions to confine programs to a limited set of resources. AppArmor can be configured for any
application to reduce its potential attack surface and provide greater in-depth defense. It is
configured through profiles tuned to allow the access needed by a specific program or container,
such as Linux capabilities, network access, file permissions, etc. Each profile can be run in either
*enforcing* mode, which blocks access to disallowed resources, or *complain* mode, which only reports
violations. 
-->
AppArmor 是一个 Linux 内核安全模块，
它补充了基于标准 Linux 用户和组的权限，将程序限制在一组有限的资源中。
AppArmor 可以配置为任何应用程序减少潜在的攻击面，并且提供更加深入的防御。
它通过调整配置文件进行配置，以允许特定程序或容器所需的访问，
如 Linux 权能字、网络访问、文件权限等。
每个配置文件都可以在
*强制（enforcing）* 模式（阻止访问不允许的资源）或
*投诉（complain）* 模式（仅报告冲突）下运行。

<!-- 
AppArmor can help you to run a more secure deployment by restricting what containers are allowed to
do, and/or provide better auditing through system logs. However, it is important to keep in mind
that AppArmor is not a silver bullet and can only do so much to protect against exploits in your
application code. It is important to provide good, restrictive profiles, and harden your
applications and cluster from other angles as well.
-->
AppArmor 可以通过限制允许容器执行的操作，
和/或通过系统日志提供更好的审计来帮助你运行更安全的部署。
但是，重要的是要记住 AppArmor 不是灵丹妙药，
只能做部分事情来防止应用程序代码中的漏洞。
提供良好的限制性配置文件，并从其他角度强化你的应用程序和集群非常重要。

## {{% heading "objectives" %}}

<!-- 
* See an example of how to load a profile on a node
* Learn how to enforce the profile on a Pod
* Learn how to check that the profile is loaded
* See what happens when a profile is violated
* See what happens when a profile cannot be loaded 
-->
* 查看如何在节点上加载配置文件示例
* 了解如何在 Pod 上强制执行配置文件
* 了解如何检查配置文件是否已加载
* 查看违反配置文件时会发生什么
* 查看无法加载配置文件时会发生什么

## {{% heading "prerequisites" %}}

<!-- Make sure: -->
确保：

<!-- 
1. Kubernetes version is at least v1.4 -- Kubernetes support for AppArmor was added in
   v1.4. Kubernetes components older than v1.4 are not aware of the new AppArmor annotations, and
   will **silently ignore** any AppArmor settings that are provided. To ensure that your Pods are
   receiving the expected protections, it is important to verify the Kubelet version of your nodes:

   ```shell
   kubectl get nodes -o=jsonpath=$'{range .items[*]}{@.metadata.name}: {@.status.nodeInfo.kubeletVersion}\n{end}'
   ```
   ```
   gke-test-default-pool-239f5d02-gyn2: v1.4.0
   gke-test-default-pool-239f5d02-x1kf: v1.4.0
   gke-test-default-pool-239f5d02-xwux: v1.4.0
   ```
-->
1. Kubernetes 版本至少是 v1.4 —— AppArmor 在 Kubernetes v1.4 版本中才添加了对 AppArmor 的支持。
   早于 v1.4 版本的 Kubernetes 组件不知道新的 AppArmor 注解
   并且将会 **默认忽略** 提供的任何 AppArmor 设置。
   为了确保你的 Pod 能够得到预期的保护，必须验证节点的 Kubelet 版本：

   ```shell
   kubectl get nodes -o=jsonpath=$'{range .items[*]}{@.metadata.name}: {@.status.nodeInfo.kubeletVersion}\n{end}'
   ```
   ```
   gke-test-default-pool-239f5d02-gyn2: v1.4.0
   gke-test-default-pool-239f5d02-x1kf: v1.4.0
   gke-test-default-pool-239f5d02-xwux: v1.4.0
   ```

<!-- 
2. AppArmor kernel module is enabled -- For the Linux kernel to enforce an AppArmor profile, the
   AppArmor kernel module must be installed and enabled. Several distributions enable the module by
   default, such as Ubuntu and SUSE, and many others provide optional support. To check whether the
   module is enabled, check the `/sys/module/apparmor/parameters/enabled` file:

   ```shell
   cat /sys/module/apparmor/parameters/enabled
   Y
   ```

   If the Kubelet contains AppArmor support (>= v1.4), it will refuse to run a Pod with AppArmor
   options if the kernel module is not enabled.

  {{< note >}}
  Ubuntu carries many AppArmor patches that have not been merged into the upstream Linux
  kernel, including patches that add additional hooks and features. Kubernetes has only been
  tested with the upstream version, and does not promise support for other features.
  {{< /note >}}
-->
2. AppArmor 内核模块已启用 —— 要使 Linux 内核强制执行 AppArmor 配置文件，
   必须安装并且启动 AppArmor 内核模块。默认情况下，有几个发行版支持该模块，
   如 Ubuntu 和 SUSE，还有许多发行版提供可选支持。要检查模块是否已启用，请检查
   `/sys/module/apparmor/parameters/enabled` 文件：

   ```shell
   cat /sys/module/apparmor/parameters/enabled
   Y
   ```

   如果 Kubelet 包含 AppArmor 支持（>= v1.4），
   但是内核模块未启用，它将拒绝运行带有 AppArmor 选项的 Pod。

  {{< note >}}
  Ubuntu 携带了许多没有合并到上游 Linux 内核中的 AppArmor 补丁，
  包括添加附加钩子和特性的补丁。Kubernetes 只在上游版本中测试过，不承诺支持其他特性。
  {{< /note >}}

<!--
3. Container runtime supports AppArmor -- Currently all common Kubernetes-supported container
   runtimes should support AppArmor, like {{< glossary_tooltip term_id="docker">}},
   {{< glossary_tooltip term_id="cri-o" >}} or {{< glossary_tooltip term_id="containerd" >}}.
   Please refer to the corresponding runtime documentation and verify that the cluster fulfills
   the requirements to use AppArmor.
-->
3. 容器运行时支持 AppArmor —— 目前所有常见的 Kubernetes 支持的容器运行时都应该支持 AppArmor，
   像 {{< glossary_tooltip term_id="docker">}}，{{< glossary_tooltip term_id="cri-o" >}}
   或 {{< glossary_tooltip term_id="containerd" >}}。
   请参考相应的运行时文档并验证集群是否满足使用 AppArmor 的要求。

<!-- 
4. Profile is loaded -- AppArmor is applied to a Pod by specifying an AppArmor profile that each
   container should be run with. If any of the specified profiles is not already loaded in the
   kernel, the Kubelet (>= v1.4) will reject the Pod. You can view which profiles are loaded on a
   node by checking the `/sys/kernel/security/apparmor/profiles` file. For example:

   ```shell
   ssh gke-test-default-pool-239f5d02-gyn2 "sudo cat /sys/kernel/security/apparmor/profiles | sort"
   ```
   ```
   apparmor-test-deny-write (enforce)
   apparmor-test-audit-write (enforce)
   docker-default (enforce)
   k8s-nginx (enforce)
   ```

   For more details on loading profiles on nodes, see
   [Setting up nodes with profiles](#setting-up-nodes-with-profiles).
-->
4. 配置文件已加载 —— 通过指定每个容器都应使用的 AppArmor 配置文件，
   AppArmor 会被应用到 Pod 上。如果指定的任何配置文件尚未加载到内核，
   Kubelet（>= v1.4） 将拒绝 Pod。
   通过检查 `/sys/kernel/security/apparmor/profiles` 文件，
   可以查看节点加载了哪些配置文件。例如:

   ```shell
   ssh gke-test-default-pool-239f5d02-gyn2 "sudo cat /sys/kernel/security/apparmor/profiles | sort"
   ```
   ```
   apparmor-test-deny-write (enforce)
   apparmor-test-audit-write (enforce)
   docker-default (enforce)
   k8s-nginx (enforce)
   ```

   有关在节点上加载配置文件的详细信息，请参见[使用配置文件设置节点](#setting-up-nodes-with-profiles)。

<!-- 
As long as the Kubelet version includes AppArmor support (>= v1.4), the Kubelet will reject a Pod
with AppArmor options if any of the prerequisites are not met. You can also verify AppArmor support
on nodes by checking the node ready condition message (though this is likely to be removed in a
later release): 
-->
只要 Kubelet 版本包含 AppArmor 支持(>=v1.4)，
如果不满足这些先决条件，Kubelet 将拒绝带有 AppArmor 选项的 Pod。
你还可以通过检查节点就绪状况消息来验证节点上的 AppArmor 支持（尽管这可能会在以后的版本中删除）：

```shell
kubectl get nodes -o=jsonpath=$'{range .items[*]}{@.metadata.name}: {.status.conditions[?(@.reason=="KubeletReady")].message}\n{end}'
```
```
gke-test-default-pool-239f5d02-gyn2: kubelet is posting ready status. AppArmor enabled
gke-test-default-pool-239f5d02-x1kf: kubelet is posting ready status. AppArmor enabled
gke-test-default-pool-239f5d02-xwux: kubelet is posting ready status. AppArmor enabled
```

<!-- lessoncontent -->

<!-- ## Securing a Pod -->
## 保护 Pod {#securing-a-pod}

{{< note >}}
<!-- 
AppArmor is currently in beta, so options are specified as annotations. Once support graduates to
general availability, the annotations will be replaced with first-class fields (more details in
[Upgrade path to GA](#upgrade-path-to-general-availability)).
-->
AppArmor 目前处于 Beta 阶段，因此选项以注解形式设定。
一旦 AppArmor 支持进入正式发布阶段，注解将被替换为一阶的资源字段
（更多详情参见[升级到 GA 的途径](#upgrade-path-to-general-availability)）。
{{< /note >}}

<!--
AppArmor profiles are specified *per-container*. To specify the AppArmor profile to run a Pod
container with, add an annotation to the Pod's metadata: 
-->
AppArmor 配置文件是按 *逐个容器* 的形式来设置的。
要指定用来运行 Pod 容器的 AppArmor 配置文件，请向 Pod 的 metadata 添加注解：

```yaml
container.apparmor.security.beta.kubernetes.io/<container_name>: <profile_ref>
```

<!-- 
Where `<container_name>` is the name of the container to apply the profile to, and `<profile_ref>`
specifies the profile to apply. The `profile_ref` can be one of: 
-->
`<container_name>` 的名称是配置文件所针对的容器的名称，`<profile_def>` 则设置要应用的配置文件。
`<profile_ref>` 可以是以下取值之一：

<!-- 
* `runtime/default` to apply the runtime's default profile
* `localhost/<profile_name>` to apply the profile loaded on the host with the name `<profile_name>`
* `unconfined` to indicate that no profiles will be loaded 
-->
* `runtime/default` 应用运行时的默认配置
* `localhost/<profile_name>` 应用在主机上加载的名为 `<profile_name>` 的配置文件
* `unconfined` 表示不加载配置文件

<!-- 
See the [API Reference](#api-reference) for the full details on the annotation and profile name formats.
-->
有关注解和配置文件名称格式的详细信息，请参阅[API 参考](#api-reference)。

<!-- 
Kubernetes AppArmor enforcement works by first checking that all the prerequisites have been
met, and then forwarding the profile selection to the container runtime for enforcement. If the
prerequisites have not been met, the Pod will be rejected, and will not run. 
-->
Kubernetes AppArmor 强制执行机制首先检查所有先决条件都已满足，
然后将所选的配置文件转发到容器运行时进行强制执行。
如果未满足先决条件，Pod 将被拒绝，并且不会运行。

<!-- 
To verify that the profile was applied, you can look for the AppArmor security option listed in the container created event: 
-->
要验证是否应用了配置文件，可以在容器创建事件中查找所列出的 AppArmor 安全选项：

```shell
kubectl get events | grep Created
```
```
22s        22s         1         hello-apparmor     Pod       spec.containers{hello}   Normal    Created     {kubelet e2e-test-stclair-node-pool-31nt}   Created container with docker id 269a53b202d3; Security:[seccomp=unconfined apparmor=k8s-apparmor-example-deny-write]
```

<!-- 
You can also verify directly that the container's root process is running with the correct profile by checking its proc attr: 
-->
你还可以通过检查容器的 proc attr，直接验证容器的根进程是否以正确的配置文件运行：

```shell
kubectl exec <pod_name> cat /proc/1/attr/current
```
```
k8s-apparmor-example-deny-write (enforce)
```

<!-- ## Example -->
## 举例 {#example}

<!-- *This example assumes you have already set up a cluster with AppArmor support.* -->
*本例假设你已经设置了一个集群使用 AppArmor 支持。*

<!-- 
First, we need to load the profile we want to use onto our nodes. This profile denies all file writes: 
-->
首先，我们需要将要使用的配置文件加载到节点上。配置文件拒绝所有文件写入：

```shell
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```

<!-- 
Since we don't know where the Pod will be scheduled, we'll need to load the profile on all our
nodes. For this example we'll use SSH to install the profiles, but other approaches are
discussed in [Setting up nodes with profiles](#setting-up-nodes-with-profiles). 
-->
由于我们不知道 Pod 将被调度到哪里，我们需要在所有节点上加载配置文件。
在本例中，我们将使用 SSH 来安装概要文件，
但是在[使用配置文件设置节点](#setting-up-nodes-with-profiles)中讨论了其他方法。

```shell
NODES=(
    # The SSH-accessible domain names of your nodes
    gke-test-default-pool-239f5d02-gyn2.us-central1-a.my-k8s
    gke-test-default-pool-239f5d02-x1kf.us-central1-a.my-k8s
    gke-test-default-pool-239f5d02-xwux.us-central1-a.my-k8s)
for NODE in ${NODES[*]}; do ssh $NODE 'sudo apparmor_parser -q <<EOF
#include <tunables/global>

profile k8s-apparmor-example-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
EOF'
done
```

<!-- Next, we'll run a simple "Hello AppArmor" pod with the deny-write profile: -->
接下来，我们将运行一个带有拒绝写入配置文件的简单 “Hello AppArmor” Pod：

{{< codenew file="pods/security/hello-apparmor.yaml" >}}

```shell
kubectl create -f ./hello-apparmor.yaml
```

<!-- 
If we look at the pod events, we can see that the Pod container was created with the AppArmor
profile "k8s-apparmor-example-deny-write": 
-->
如果我们查看 Pod 事件，我们可以看到 Pod 容器是用 AppArmor
配置文件 “k8s-apparmor-example-deny-write” 所创建的：

```shell
kubectl get events | grep hello-apparmor
```
```
14s        14s         1         hello-apparmor   Pod                                Normal    Scheduled   {default-scheduler }                           Successfully assigned hello-apparmor to gke-test-default-pool-239f5d02-gyn2
14s        14s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Pulling     {kubelet gke-test-default-pool-239f5d02-gyn2}   pulling image "busybox"
13s        13s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Pulled      {kubelet gke-test-default-pool-239f5d02-gyn2}   Successfully pulled image "busybox"
13s        13s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Created     {kubelet gke-test-default-pool-239f5d02-gyn2}   Created container with docker id 06b6cd1c0989; Security:[seccomp=unconfined apparmor=k8s-apparmor-example-deny-write]
13s        13s         1         hello-apparmor   Pod       spec.containers{hello}   Normal    Started     {kubelet gke-test-default-pool-239f5d02-gyn2}   Started container with docker id 06b6cd1c0989
```

<!-- We can verify that the container is actually running with that profile by checking its proc attr: -->
我们可以通过检查该配置文件的 proc attr 来验证容器是否实际使用该配置文件运行：

```shell
kubectl exec hello-apparmor -- cat /proc/1/attr/current
```
```
k8s-apparmor-example-deny-write (enforce)
```

<!-- Finally, we can see what happens if we try to violate the profile by writing to a file: -->
最后，我们可以看到，如果我们尝试通过写入文件来违反配置文件会发生什么：

```shell
kubectl exec hello-apparmor -- touch /tmp/test
```
```
touch: /tmp/test: Permission denied
error: error executing remote command: command terminated with non-zero exit code: Error executing in Docker Container: 1
```

<!-- To wrap up, let's look at what happens if we try to specify a profile that hasn't been loaded: -->
最后，让我们看看如果我们试图指定一个尚未加载的配置文件会发生什么：

```shell
kubectl create -f /dev/stdin <<EOF
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor-2
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/k8s-apparmor-example-allow-write
spec:
  containers:
  - name: hello
    image: busybox:1.28
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
EOF
pod/hello-apparmor-2 created
```

```shell
kubectl describe pod hello-apparmor-2
```
```
Name:          hello-apparmor-2
Namespace:     default
Node:          gke-test-default-pool-239f5d02-x1kf/
Start Time:    Tue, 30 Aug 2016 17:58:56 -0700
Labels:        <none>
Annotations:   container.apparmor.security.beta.kubernetes.io/hello=localhost/k8s-apparmor-example-allow-write
Status:        Pending
Reason:        AppArmor
Message:       Pod Cannot enforce AppArmor: profile "k8s-apparmor-example-allow-write" is not loaded
IP:
Controllers:   <none>
Containers:
  hello:
    Container ID:
    Image:     busybox
    Image ID:
    Port:
    Command:
      sh
      -c
      echo 'Hello AppArmor!' && sleep 1h
    State:              Waiting
      Reason:           Blocked
    Ready:              False
    Restart Count:      0
    Environment:        <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-dnz7v (ro)
Conditions:
  Type          Status
  Initialized   True
  Ready         False
  PodScheduled  True
Volumes:
  default-token-dnz7v:
    Type:    Secret (a volume populated by a Secret)
    SecretName:    default-token-dnz7v
    Optional:   false
QoS Class:      BestEffort
Node-Selectors: <none>
Tolerations:    <none>
Events:
  FirstSeen    LastSeen    Count    From                        SubobjectPath    Type        Reason        Message
  ---------    --------    -----    ----                        -------------    --------    ------        -------
  23s          23s         1        {default-scheduler }                         Normal      Scheduled     Successfully assigned hello-apparmor-2 to e2e-test-stclair-node-pool-t1f5
  23s          23s         1        {kubelet e2e-test-stclair-node-pool-t1f5}             Warning        AppArmor    Cannot enforce AppArmor: profile "k8s-apparmor-example-allow-write" is not loaded
```

<!-- 
Note the pod status is Pending, with a helpful error message: `Pod Cannot enforce AppArmor: profile
"k8s-apparmor-example-allow-write" is not loaded`. An event was also recorded with the same message. 
-->
注意 Pod 呈现 Pending 状态，并且显示一条有用的错误信息：
`Pod Cannot enforce AppArmor: profile "k8s-apparmor-example-allow-write" is not loaded`。
还用相同的消息记录了一个事件。

<!-- ## Administration -->
## 管理 {#administration}

<!-- ### Setting up nodes with profiles -->
### 使用配置文件设置节点 {#setting-up-nodes-with-profiles}

<!-- 
Kubernetes does not currently provide any native mechanisms for loading AppArmor profiles onto
nodes. There are lots of ways to setup the profiles though, such as: 
-->
Kubernetes 目前不提供任何本地机制来将 AppArmor 配置文件加载到节点上。
有很多方法可以设置配置文件，例如：

<!-- 
* Through a [DaemonSet](/docs/concepts/workloads/controllers/daemonset/) that runs a Pod on each node to
  ensure the correct profiles are loaded. An example implementation can be found
  [here](https://git.k8s.io/kubernetes/test/images/apparmor-loader).
* At node initialization time, using your node initialization scripts (e.g. Salt, Ansible, etc.) or
  image.
* By copying the profiles to each node and loading them through SSH, as demonstrated in the
  [Example](#example). 
-->
* 通过在每个节点上运行 Pod 的
  [DaemonSet](/zh-cn/docs/concepts/workloads/controllers/daemonset/)来确保加载了正确的配置文件。
  可以在[这里](https://git.k8s.io/kubernetes/test/images/apparmor-loader)找到实现示例。
* 在节点初始化时，使用节点初始化脚本(例如 Salt、Ansible 等)或镜像。
* 通过将配置文件复制到每个节点并通过 SSH 加载它们，如[示例](#example)。

<!-- 
The scheduler is not aware of which profiles are loaded onto which node, so the full set of profiles
must be loaded onto every node.  An alternative approach is to add a node label for each profile (or
class of profiles) on the node, and use a
[node selector](/docs/concepts/configuration/assign-pod-node/) to ensure the Pod is run on a
node with the required profile. 
-->
调度程序不知道哪些配置文件加载到哪个节点上，因此必须将全套配置文件加载到每个节点上。
另一种方法是为节点上的每个配置文件（或配置文件类）添加节点标签，
并使用[节点选择器](/zh-cn/docs/concepts/configuration/assign-pod-node/)确保
Pod 在具有所需配置文件的节点上运行。

<!-- ### Restricting profiles with the PodSecurityPolicy -->
### 使用 PodSecurityPolicy 限制配置文件 {#restricting-profiles-with-the-podsecuritypolicy}

{{< note >}}
<!-- 
PodSecurityPolicy is deprecated in Kubernetes v1.21, and will be removed in v1.25.
See [PodSecurityPolicy](/docs/concepts/security/pod-security-policy/) documentation for more information.
-->
PodSecurityPolicy 在 Kubernetes v1.21 版本中已被废弃，将在 v1.25 版本移除。
查看 [PodSecurityPolicy](/zh-cn/docs/concepts/security/pod-security-policy/) 文档获取更多信息。
{{< /note >}}

<!-- 
If the PodSecurityPolicy extension is enabled, cluster-wide AppArmor restrictions can be applied. To
enable the PodSecurityPolicy, the following flag must be set on the `apiserver`: 
-->
如果启用了 PodSecurityPolicy 扩展，则可以应用集群范围的 AppArmor 限制。
要启用 PodSecurityPolicy，必须在 `apiserver` 上设置以下标志：

```
--enable-admission-plugins=PodSecurityPolicy[,others...]
```

<!-- The AppArmor options can be specified as annotations on the PodSecurityPolicy: -->
AppArmor 选项可以指定为 PodSecurityPolicy 上的注解：

```yaml
apparmor.security.beta.kubernetes.io/defaultProfileName: <profile_ref>
apparmor.security.beta.kubernetes.io/allowedProfileNames: <profile_ref>[,others...]
```

<!-- 
The default profile name option specifies the profile to apply to containers by default when none is
specified. The allowed profile names option specifies a list of profiles that Pod containers are
allowed to be run with. If both options are provided, the default must be allowed. The profiles are
specified in the same format as on containers. See the [API Reference](#api-reference) for the full
specification. 
-->
默认配置文件名选项指定默认情况下在未指定任何配置文件时应用于容器的配置文件。
所允许的配置文件名称选项指定允许 Pod 容器运行期间所对应的配置文件列表。
如果同时提供了这两个选项，则必须允许默认值。
配置文件的指定格式与容器上的相同。有关完整规范，请参阅 [API 参考](#api-reference)。

<!-- ### Disabling AppArmor -->
### 禁用 AppArmor {#disabling-apparmor}

<!-- If you do not want AppArmor to be available on your cluster, it can be disabled by a command-line flag: -->
如果你不希望 AppArmor 在集群上可用，可以通过命令行标志禁用它：

```
--feature-gates=AppArmor=false
```

<!-- 
When disabled, any Pod that includes an AppArmor profile will fail validation with a "Forbidden"
error. 
-->
禁用时，任何包含 AppArmor 配置文件的 Pod 都将导致验证失败，且返回 “Forbidden” 错误。

{{<note>}}
<!--
Even if the Kubernetes feature is disabled, runtimes may still enforce the default profile. The
option to disable the AppArmor feature will be removed when AppArmor graduates to general
availability (GA).
-->
即使此 Kubernetes 特性被禁用，运行时仍可能强制执行默认配置文件。
当 AppArmor 升级为正式版 (GA) 时，禁用 AppArmor 功能的选项将被删除。

{{</note>}}


<!-- ## Authoring Profiles -->
## 编写配置文件 {#authoring-profiles}

<!-- 
Getting AppArmor profiles specified correctly can be a tricky business. Fortunately there are some
tools to help with that: 
-->
获得正确指定的 AppArmor 配置文件可能是一件棘手的事情。幸运的是，有一些工具可以帮助你做到这一点：

<!-- 
* `aa-genprof` and `aa-logprof` generate profile rules by monitoring an application's activity and
  logs, and admitting the actions it takes. Further instructions are provided by the
  [AppArmor documentation](https://gitlab.com/apparmor/apparmor/wikis/Profiling_with_tools).
* [bane](https://github.com/jfrazelle/bane) is an AppArmor profile generator for Docker that uses a
  simplified profile language. 
-->
* `aa-genprof` 和 `aa-logprof`
  通过监视应用程序的活动和日志并准许它所执行的操作来生成配置文件规则。
  [AppArmor 文档](https://gitlab.com/apparmor/apparmor/wikis/Profiling_with_tools)提供了进一步的指导。
* [bane](https://github.com/jfrazelle/bane)
  是一个用于 Docker的 AppArmor 配置文件生成器，它使用一种简化的画像语言（profile language）

<!-- 
To debug problems with AppArmor, you can check the system logs to see what, specifically, was
denied. AppArmor logs verbose messages to `dmesg`, and errors can usually be found in the system
logs or through `journalctl`. More information is provided in
[AppArmor failures](https://gitlab.com/apparmor/apparmor/wikis/AppArmor_Failures). 
-->
想要调试 AppArmor 的问题，你可以检查系统日志，查看具体拒绝了什么。
AppArmor 将详细消息记录到 `dmesg`，
错误通常可以在系统日志中或通过 `journalctl` 找到。
更多详细信息见 [AppArmor 失败](https://gitlab.com/apparmor/apparmor/wikis/AppArmor_Failures)。

<!-- ## API Reference -->
## API 参考 {#api-reference}

<!-- ### Pod Annotation -->
### Pod 注解 {#pod-annotation}

<!-- Specifying the profile a container will run with: -->
指定容器将使用的配置文件：

<!-- 
- **key**: `container.apparmor.security.beta.kubernetes.io/<container_name>`
  Where `<container_name>` matches the name of a container in the Pod.
  A separate profile can be specified for each container in the Pod.
- **value**: a profile reference, described below 
-->
- **键名**: `container.apparmor.security.beta.kubernetes.io/<container_name>`
  ，其中 `<container_name>` 与 Pod 中某容器的名称匹配。
  可以为 Pod 中的每个容器指定单独的配置文件。
- **键值**: 对配置文件的引用，如下所述

<!-- ### Profile Reference -->
### 配置文件引用 {#profile-reference}

<!-- 
- `runtime/default`: Refers to the default runtime profile.
  - Equivalent to not specifying a profile (without a PodSecurityPolicy default), except it still
    requires AppArmor to be enabled.
  - In practice, many container runtimes use the same OCI default profile, defined here:
    https://github.com/containers/common/blob/main/pkg/apparmor/apparmor_linux_template.go
- `localhost/<profile_name>`: Refers to a profile loaded on the node (localhost) by name.
  - The possible profile names are detailed in the
    [core policy reference](https://gitlab.com/apparmor/apparmor/wikis/AppArmor_Core_Policy_Reference#profile-names-and-attachment-specifications).
- `unconfined`: This effectively disables AppArmor on the container. 
-->
- `runtime/default`: 指默认运行时配置文件。
  - 等同于不指定配置文件（没有 PodSecurityPolicy 默认值），只是它仍然需要启用 AppArmor。
  - 实际上，许多容器运行时使用相同的 OCI 默认配置文件，在此处定义：
    https://github.com/containers/common/blob/main/pkg/apparmor/apparmor_linux_template.go
- `localhost/<profile_name>`: 按名称引用加载到节点（localhost）上的配置文件。
  - 可能的配置文件名在[核心策略参考](https://gitlab.com/apparmor/apparmor/wikis/AppArmor_Core_Policy_Reference#profile-names-and-attachment-specifications)。
- `unconfined`: 这相当于为容器禁用 AppArmor。

<!-- Any other profile reference format is invalid. -->
任何其他配置文件引用格式无效。

<!-- ### PodSecurityPolicy Annotations -->
### PodSecurityPolicy 注解 {#podsecuritypolicy-annotations}

<!-- Specifying the default profile to apply to containers when none is provided: -->
指定在未提供容器时应用于容器的默认配置文件：

<!-- 
* **key**: `apparmor.security.beta.kubernetes.io/defaultProfileName`
* **value**: a profile reference, described above 
-->
* **键名**: `apparmor.security.beta.kubernetes.io/defaultProfileName`
* **键值**: 如上述文件参考所述

<!-- Specifying the list of profiles Pod containers is allowed to specify: -->
上面描述的指定配置文件，Pod 容器列表的配置文件引用允许指定：

<!-- 
* **key**: `apparmor.security.beta.kubernetes.io/allowedProfileNames`
* **value**: a comma-separated list of profile references (described above)
  - Although an escaped comma is a legal character in a profile name, it cannot be explicitly
    allowed here. 
-->
* **键名**: `apparmor.security.beta.kubernetes.io/allowedProfileNames`
* **键值**: 配置文件引用的逗号分隔列表（如上所述）
  - 尽管转义逗号是配置文件名中的合法字符，但此处不能显式允许。

## {{% heading "whatsnext" %}}

<!-- Additional resources: -->
其他资源：

<!-- 
* [Quick guide to the AppArmor profile language](https://gitlab.com/apparmor/apparmor/wikis/QuickProfileLanguage)
* [AppArmor core policy reference](https://gitlab.com/apparmor/apparmor/wikis/Policy_Layout) 
-->
* [Apparmor 配置文件语言快速指南](https://gitlab.com/apparmor/apparmor/wikis/QuickProfileLanguage)
* [Apparmor 核心策略参考](https://gitlab.com/apparmor/apparmor/wikis/Policy_Layout)
