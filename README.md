# Playsafe 動力安全作業

## 整體架構

![](https://raw.githubusercontent.com/patrick55529617/playsafework/refs/heads/main/pics/0-system-structure.png)

說明:

1. 用 KIND 建置一個 k8s cluster，並由 3個 master node & 4個 worker node 所組成
2. 配置 2個 infra node 及 2個 app node
3. 整個 k8s 環境 expose 一個 30000 port 給外部服務
4. 在 k8s 內部建置一個 prometheus 服務進行監控
5. 在 k8s 環境外部安裝一個 grafana docker，並設定 network 為 `kind`，使其可與 k8s prometheus 溝通
6. 簡單建立一個 app 叫做 hello world，並幫其建立一個 hpa，memory 抵達 50% 後會水平拓展 pod

## 專案結構

1. create_cluster: KIND cluster 建立的 config
2. install_metallb: 安裝 metallb 的 config
3. build_monitor: 建置 prometheus + grafana
4. hpa: 建置測試用 container image + hpa

## 詳細步驟說明

### 1. 安裝 KIND & 建立 cluster

- 1. 安裝 KIND (本次作業在 macOS 環境下進行，docker 已預先安裝完畢)
```
[ $(uname -m) = arm64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.24.0/kind-darwin-arm64
chmod +x ./kind
mv ./kind /usr/local/bin/kind
```
- 2. 建置 cluster
```
kind create cluster --name playsafe --config create_cluster/config.yaml
kubectl cluster-info --context playsafe-kind
kubectl get nodes -o wide
```
![](https://raw.githubusercontent.com/patrick55529617/playsafework/refs/heads/main/pics/1-getnodes.png)

### 2. 安裝 metallb L2 mode

```
# Install CRD
kubectl apply -f install_metallb/metallb.yaml

# Set up metallb ip range
kubectl apply -f install_metallb/metallb-config.yaml
```

### 3. 安裝 Prometheus, node exporter, kube-state-metrics 

```
# 這裡使用 helm chart 進行 Prometheus 安裝，helm 已預先安裝完畢
# 下載 Prometheus helm chart
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# config 已存放在 repo 內，直接使用，裡面有設定了 loadbalancer 跟 nodeport 待會使用
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  -f build_monitor/prometheus-values.yaml \
  --create-namespace \
```

### 4. 安裝 Grafana 在 kind cluster 外部

```
# 安裝 grafana 並設定 network 與 kind 共用
docker run -d -p 3000:3000 --network kind --name=grafana grafana/grafana
```

1. 進 localhost:3000 使用 admin 登入
2. 到 data resource 新增 prometheus，並設定 ip 為 http://NODE_IP:30000
3. 載入 build_monitor/grafana-dashboard.yaml 的設定，即可呈現設定好的 dashboard

![](https://raw.githubusercontent.com/patrick55529617/playsafework/refs/heads/main/pics/2-grafana.png)

### 5. 安裝一個簡易的 app 並設定 hpa

```
# 安裝 example image
kubectl apply -f hpa/whale.yaml

# 安裝 hpa
kubectl apply -f hpa/hpa.yaml
```

![](https://raw.githubusercontent.com/patrick55529617/playsafework/refs/heads/main/pics/3-whale.png)

![](https://raw.githubusercontent.com/patrick55529617/playsafework/refs/heads/main/pics/4-hpa.png)
