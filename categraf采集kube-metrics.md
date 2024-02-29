## categraf 安装
### 版本v0.3.47
k8s节点：
```
wget https://codeload.github.com/flashcatcloud/categraf/zip/refs/heads/main
mv main main.zip
kubectl create namespace monitoring
kubectl apply -n monitoring -f k8s/daemonset.yaml
```
修改配置使其能连接到夜莺: https://flashcat.cloud/docs/content/flashcat-monitor/categraf/3-configuration/

## kube-metrics 安装
k8s节点:
```
wget https://github.com/kubernetes/kube-state-metrics/archive/refs/tags/v2.10.1.zip
unzip v2.10.1.zip
cd kube-state-metrics-2.10.1/
cd examples/
kubectl apply -f standard/
```

### categraf 采集 kube-metrics
categraf修改: configmap
            input-kubelet-metrics
             prometheus.toml
```
# collect interval
interval = 15

[[instances]]
urls = ["http://127.0.0.1:10249/metrics"]
labels = { job="kube-proxy" }
[[instances]]
urls = ["https://127.0.0.1:10250/metrics"]
bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
use_tls = true
insecure_skip_verify = true
labels = { job="kubelet" }
[[instances]]
urls = ["https://127.0.0.1:10250/metrics/cadvisor"]
bearer_token_file = "/var/run/secrets/kubernetes.io/serviceaccount/token"
## 这里的token地址就是我们上面base64转码后把token放在的那个文件里面的路径
use_tls = true
insecure_skip_verify = true
labels = { job="cadvisor" }
[[instances]]                             ##这里开始添加
urls = ["http://kube-state-metrics.kube-system:8080/metrics"]        
use_tls = true
insecure_skip_verify = true
labels = { job="kube-state-metrics" }
[[instances]]
urls = ["http://kube-state-metrics.kube-system:8081/metrics"]
use_tls = true
insecure_skip_verify = true
labels = { job="kube-state-metrics-self" }
```
