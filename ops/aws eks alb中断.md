### eks 使用 alb ingress，服务更新会出现较长时间的中断
#### 一个解决办法就是,启用 Readiness Gate  
只需要在 namespace 中添加一个 label:
```
$ kubectl create namespace readiness
namespace/readiness created
$ kubectl label namespace readiness elbv2.k8s.aws/pod-readiness-gate-inject=enabled
namespace/readiness labeled

$ kubectl describe namespace readiness
Name:         readiness
Labels:       elbv2.k8s.aws/pod-readiness-gate-inject=enabled
Annotations:  <none>
Status:       Active
```
但是经过本人测试，还是会出现中断，虽然中断时间减少了，那就：  
#### 配合k8s生命周期使用
因为 k8s prestop是同步执行的，在k8s prestop中进行timesleep可以减慢旧pod的terminating速度，给这破alb更多的切换时间，从而缩短中断时间。
