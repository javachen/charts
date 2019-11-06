# Helm Charts使用说明


> 注意，所有命令需要在当前目录执行

## 通过rook部署Ceph持久化存储

### 部署rook

    helm repo add rook-release https://charts.rook.io/release
    
    
    helm install --name rook-ceph --namespace rook-ceph rook-release/rook-ceph
    
    kubectl --namespace rook-ceph get pods -l "app=rook-ceph-operator"

### 通过rook部署集群

> 在部署集群时，各节点上需要至少有1块硬盘，且节点数量大于3个。

参考

https://blog.51cto.com/bigboss/2320016

https://blog.csdn.net/networken/article/details/85772418

https://blog.csdn.net/aixiaoyang168/article/details/86467931

```bash
git clone https://github.com/rook/rook
cd rook/cluster/examples/kubernetes/ceph

cat > cluster-minimal.yaml << EOF
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: ceph/ceph:v14.2.4-20190917
    allowUnsupported: false
  dataDirHostPath: /var/lib/rook
  mon:
    count: 1
    allowMultiplePerNode: false
  dashboard:
    enabled: true
    ssl: true
  monitoring:
    enabled: false  # requires Prometheus to be pre-installed
    rulesNamespace: rook-ceph
  network:
    hostNetwork: false
  storage:
    useAllNodes: false
    useAllDevices: false
    deviceFilter:
    location:
    config:
      nodes:
      - name: "k8s-rke-node001"
        devices:
        - name: "sdb"  
EOF

kubectl apply -f cluster-minimal.yaml

kubectl get pods -n rook-ceph -w
kubectl -n rook-ceph get deployment
kubectl get service -n rook-ceph

kubectl apply -f dashboard-external-https.yaml
kubectl get service -n rook-ceph | grep dashboard
MGR_POD=`kubectl get pod -n rook-ceph | grep mgr | awk '{print $1}'`
kubectl -n rook-ceph logs $MGR_POD | grep password
# https://192.168.56.111:30225/

cat > storageclass-test.yaml << EOF
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 1
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  blockPool: replicapool
  clusterNamespace: rook-ceph
  fstype: xfs
EOF

kubectl apply -f storageclass-test.yaml
kubectl get storageclass

kubectl apply -f toolbox.yaml
kubectl -n rook-ceph get pods -o wide | grep ceph-tools
kubectl -n rook-ceph exec -it rook-ceph-tools-856c5bc6b4-rnzxp bash
ceph status
cd /etc/ceph/
cat ceph.conf 
cat keyring
cat rbdmap 

cd ..
kubectl apply -f mysql.yaml
kubectl apply -f wordpress.yaml


kubectl delete -f cluster-minimal.yaml
kubectl delete -f dashboard-external-https.yaml
kubectl delete -f storageclass-test.yaml
kubectl delete -f toolbox.yaml
kubectl delete -f mysql.yaml
kubectl delete -f wordpress.yaml
```

## 部署elasticsearch

在我的环境中，我使用ceph来存储elasticsearch的数据，ceph持久化存储的storageClass为rook-ceph-block，如果你的环境与此不同，请根据你的环境在elasticsearch/values.yaml中变更storageClassName的值。

    helm install --name elasticsearch elasticsearch --namespace logging

### 更新elasticsearch配置

    helm upgrade elasticsearch elasticsearch --namespace logging

### 查看elasticsearch部署情况

    kubectl get pvc -n logging
    kubectl get pods -n logging

## 部署kibana

需要等待elasticsearch部署完成后再部署kibana。

    helm install --name kibana kibana --namespace logging

### 更新kibana配置

    helm upgrade kibana kibana --namespace logging

### 查看kibana部署情况

    kubectl get pods -n logging

## 部署Fluentd

Fluentd用于收集k8s产生的日志到elasticsearch中。

    helm install --name fluentd fluentd-elasticsearch -f fluentd-elasticsearch/values-test.yaml --namespace logging

### 更新Fluentd配置

    helm upgrade fluentd fluentd-elasticsearch -f fluentd-elasticsearch/values-test.yaml --namespace logging

## 部署Jaeger链路追踪系统

   Jaeger部署依赖于elasticsearch，在部署前需要先确保elasticsearch服务是可以的，可以根据你自己的环境在jaeger-operator/values-test.yaml中修改elasticsearch的地址。

   helm install --name jaeger jaeger-operator -f jaeger-operator/values-test.yaml --namespace tracing

### 更新Jaeger配置

   helm upgrade jaeger jaeger-operator -f jaeger-operator/values-test.yaml --namespace tracing

