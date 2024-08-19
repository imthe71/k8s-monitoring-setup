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



# 安裝 Kube-State-Metrics helm install kube-state-metrics prometheus-community/kube-state-metrics # 添加 Prometheus Helm Chart 來源 helm repo add prometheus-community https://prometheus-community.github.io/helm-charts helm repo update # 安裝 Prometheus helm install prometheus prometheus-community/prometheus


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

