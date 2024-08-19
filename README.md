UBUNTU 18

安裝 Docker/kubectl 
echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable" | sudo tee /etc/apt/sources.list.d/docker.list
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo snap install kubectl --classic


安裝 Kind 並創建 Kubernetes 叢集
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.18.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

創建一個配置文件 kind-config.yaml
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
EOF
使用這個配置文件來創建叢集
kind create cluster --config kind-config.yaml



# 安裝 Kube-State-Metrics 
helm install kube-state-metrics prometheus-community/kube-state-metrics 
# 添加 Prometheus Helm Chart 來源 
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts helm repo update 
# 安裝 Prometheus 
helm install prometheus prometheus-community/prometheus


vi node-exporter-daemonset.yaml



apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: node-exporter
  name: node-exporter
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: node-exporter
  template:
    metadata:
      labels:
        k8s-app: node-exporter
    spec:
      containers:
      - args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --collector.filesystem.ignored-mount-points
        - ^/(dev|proc|sys|var/lib/docker/.+)($|/)
        image: quay.io/prometheus/node-exporter:v1.5.0
        name: node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: scrape
        resources:
          limits:
            cpu: 100m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsGroup: 65534
          runAsNonRoot: true
          runAsUser: 65534
        volumeMounts:
        - mountPath: /host/proc
          name: proc
          readOnly: true
        - mountPath: /host/sys
          name: sys
          readOnly: true
      hostNetwork: true
      hostPID: true
      volumes:
      - hostPath:
          path: /proc
        name: proc
      - hostPath:
          path: /sys
        name: sys

kubectl apply -f node-exporter-daemonset.yaml
kubectl get pods -n kube-system -l k8s-app=node-exporter



root@ubuntu:~# docker network connect kind f92243eb8169
root@ubuntu:~# docker ps
CONTAINER ID   IMAGE                                COMMAND                  CREATED       STATUS       PORTS                                       NAMES
f92243eb8169   grafana/grafana                      "/run.sh"                6 hours ago   Up 6 hours   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   grafana
22aaa0145768   kindest/haproxy:v20230227-d46f45b6   "haproxy -sf 7 -W -d…"   6 hours ago   Up 6 hours   127.0.0.1:46629->6443/tcp                   kind-external-load-balancer
ad4fbbf3cedc   kindest/node:v1.26.3                 "/usr/local/bin/entr…"   6 hours ago   Up 6 hours   127.0.0.1:37339->6443/tcp                   kind-control-plane3
ff14f159bafa   kindest/node:v1.26.3                 "/usr/local/bin/entr…"   6 hours ago   Up 6 hours                                               kind-worker
8a0f114595dd   kindest/node:v1.26.3                 "/usr/local/bin/entr…"   6 hours ago   Up 6 hours   127.0.0.1:45091->6443/tcp                   kind-control-plane2
44816d2a86a9   kindest/node:v1.26.3                 "/usr/local/bin/entr…"   6 hours ago   Up 6 hours   127.0.0.1:33093->6443/tcp                   kind-control-plane
549d28f8b0f6   kindest/node:v1.26.3                 "/usr/local/bin/entr…"   6 hours ago   Up 6 hours                                               kind-worker2

查看NODE IP
kubectl get nodes -o wide

HPA實驗
1. 部署一個範例應用程式
我們將部署一個簡單的 NGINX 應用程式，然後為其創建 HPA。
1.1 創建 NGINX Deployment
首先，在 Kind 叢集中創建一個 NGINX Deployment：


kubectl create deployment nginx --image=nginx

這將創建一個 NGINX Deployment，默認情況下會創建一個 Pod。
安裝metrics-server.
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

1.2 暴露 Deployment 服務
接下來，我們將為這個 Deployment 創建一個服務，以便可以從外部訪問 NGINX：


kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort

這將創建一個 NodePort 服務，允許從集群外部通過節點 IP 訪問 NGINX。
2. 創建 HPA (Horizontal Pod Autoscaler)
現在，我們為這個 NGINX Deployment 創建一個 HPA，當 CPU 使用率超過 50% 時自動擴展 Pod，最多可擴展到 10 個 Pod。
2.1 創建 HPA
使用以下命令創建 HPA：


kubectl autoscale deployment nginx --cpu-percent=50 --min=1 --max=10

這條命令的含義是：
當 CPU 使用率超過 50% 時，HPA 將自動擴展 Pod 數量。
最少保持 1 個 Pod，最多可以擴展到 10 個 Pod。
3. 驗證 HPA
可以使用以下命令來檢查 HPA 的狀態：


kubectl get hpa

這將顯示當前 HPA 的目標 CPU 使用率、當前使用率、以及目前的 Pod 數量等信息。
4. 模擬高負載（可選）
為了測試 HPA 是否正常工作，你可以在 NGINX 部署上施加一些 CPU 負載。例如，可以運行一個簡單的負載生成器來模擬高 CPU 使用率：


kubectl run -i --tty load-generator --image=busybox /bin/sh

進入 BusyBox 容器後，使用 while 迴圈生成負載：


while true; do wget -q -O- http://nginx.default.svc.cluster.local; done

這會不斷地向 NGINX 發送請求，從而增加 CPU 使用率，觸發 HPA 的 Pod 擴展。
5. 清理資源
測試完成後，你可以刪除這些資源：


kubectl delete hpa nginx
kubectl delete deployment nginx
kubectl delete service nginx
確認 metrics-server 是否已經正常啟動並運行
kubectl get pods -n kube-system

kubectl logs -n kube-system metrics-server-6964989f78-r8n88

E0819 14:23:38.812138 1 scraper.go:149] "Failed to scrape node" err="Get \"https://172.18.0.2:10250/metrics/resource\": tls: failed to verify certificate: x509: cannot validate certificate for 172.18.0.2 because it doesn't contain any IP SANs" node="kind-control-plane3"

與節點之間的 TLS 證書驗證失敗。這個問題的根本原因是 metrics-server 嘗試使用節點的 IP 地址進行 TLS 連接，但節點證書中缺少 IP SAN（Subject Alternative Name）

編輯 metrics-server 部署
kubectl edit deployment metrics-server -n kube-system

在編輯器中找到 args 部分，並添加 --kubelet-insecure-tls 參數
spec:
  containers:
  - args:
    - --cert-dir=/tmp
    - --secure-port=10250
    - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
    - --kubelet-use-node-status-port
    - --kubelet-insecure-tls

驗證 metrics-server
kubectl get pods -n 
kube-system kubectl top nodes


root@ubuntu:~# kubectl get hpa
NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
nginx   Deployment/nginx   <unknown>/50%   1         10        1          18m
root@ubuntu:~# kubectl describe deployment nginx
Name:                   nginx
Namespace:              default
CreationTimestamp:      Mon, 19 Aug 2024 14:13:22 +0000
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:         nginx
    Port:          <none>
    Host Port:     <none>
    Environment:   <none>
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-748c667d99 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  19m   deployment-controller  Scaled up replica set nginx-748c667d99 to 1

這個 NGINX Deployment 沒有配置任何 CPU 資源請求 (resources.requests.cpu)。這會導致 HPA 無法計算 CPU 使用率，從而無法觸發擴展。

kubectl edit deployment nginx
1. 編輯 NGINX Deployment
使用以下命令編輯 Deployment：

kubectl edit deployment nginx

2. 添加資源請求
在編輯器中，找到 containers 部分，並添加 resources 配置，類似於以下內容：

spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"

這個配置會為每個 NGINX Pod 請求 100 毫核 (100m) 的 CPU 資源。
3. 保存並退出
編輯完成後，保存並退出編輯器。Kubernetes 會自動更新 Deployment 並重新創建 Pod。
4. 驗證更新
你可以使用以下命令查看更新後的 Deployment 來確保配置已經生效：

kubectl describe deployment nginx

root@ubuntu:~# kubectl get deployment nginx
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   2/2     2            2           31m


root@ubuntu:~# kubectl describe hpa nginx
Name:                                                  nginx
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Mon, 19 Aug 2024 14:13:43 +0000
Reference:                                             Deployment/nginx
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  69% (69m) / 50%
Min replicas:                                          1
Max replicas:                                          10
Deployment pods:                                       2 current / 2 desired
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
Events:
  Type     Reason                        Age                   From                       Message
  ----     ------                        ----                  ----                       -------
  Warning  FailedComputeMetricsReplicas  25m (x12 over 27m)    horizontal-pod-autoscaler  invalid metrics (1 invalid out of 1), first error is: failed to get cpu resource metric value: failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)
  Warning  FailedGetResourceMetric       22m (x21 over 27m)    horizontal-pod-autoscaler  failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)
  Warning  FailedGetResourceMetric       17m (x5 over 18m)     horizontal-pod-autoscaler  failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server is currently unable to handle the request (get pods.metrics.k8s.io)
  Warning  FailedGetResourceMetric       7m55s (x28 over 14m)  horizontal-pod-autoscaler  failed to get cpu utilization: missing request for cpu in container nginx of Pod nginx-748c667d99-5nqph
  Normal   SuccessfulRescale             39s                   horizontal-pod-autoscaler  New size: 2; reason: cpu resource utilization (percentage of request) above target

成功長出第二台

