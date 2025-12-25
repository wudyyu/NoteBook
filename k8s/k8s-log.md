## Kubernetes日志常用命令

- 1.从特定的Pod中尾随日志：
```
kubectl logs -f <pod-name>
```
> 此命令将流式传输并显示指定Pod的日志。您可以包含多个Pod名称以同时尾随多个Pod的日志


- 2.从与标签选择器匹配的Pod中尾随日志：
```
kubectl logs -f -l <label-selector>
```
> 此命令将流式传输并显示与指定的标签选择器匹配的所有Pod的日志。

- 3.从特定命名空间中的所有Pod中尾随日志：
```
kubectl logs -f -n <namespace>
```
> 此命令将流式传输并显示指定命名空间中所有Pod的日志。


- 4.使用通配符表达式从多个Pod中尾随日志
```
kubectl logs -f <pod-name-pattern>
```
> 此命令允许您指定多个Pod名称或使用通配符表达式从多个Pod中尾随日志。例如:
```
kubectl logs -f my-app-*
```
> 这将从以“my-app-”开头的所有Pod中尾随日志。

- 5.在日志输出中包含时间戳：
```
kubectl logs -f --timestamps <pod-name>
```
> 此命令将显示带有时间戳的日志，以便您可以查看日志消息生成的时间。

- 在使用kubetail时，您还可以通过指定其Pod名称来查看Kubernetes控制平面组件的日志。下面是如何查看一些常用控制平面组件的日志
> 1.查看Kubernetes API服务器的日志：
```
kubectl logs -f -n kube-system <api-server-pod-name>
```
> 将替换为API服务器Pod所在的节点的实际名称。此命令将尾随API服务器的日志。

> 2.查看Kubernetes控制器管理器的日志：
```
kubectl logs -f -n kube-system <controller-manager-pod-name>
```
> 将替换为控制器管理器Pod所在的节点的名称。此命令将尾随控制器管理器的日志。

> 3.查看Kubernetes调度器的日志
```
kubectl logs -f -n kube-system <scheduler-pod-name>
```
> 将替换为调度器Pod所在的节点的名称。此命令将尾随调度器的日志。

> 4.查看etcd集群的日志：
```
kubectl logs -f -n kube-system <etcd-pod-name>
```
> 将替换为etcd Pod所在的节点的名称。此命令将尾随etcd集群的日志。

> 记得用实际运行相应控制平面组件Pod的节点名称替换。-n kube-system标志用于指定控制平面组件通常部署的命名空间。
> 通过 tail 这些控制平面组件的日志，您可以监视它们的活动，检测任何问题或错误，并深入了解您的Kubernetes集群的行为

