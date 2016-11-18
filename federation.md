# 部署
(github)[https://github.com/kelseyhightower/kubernetes-cluster-federation]

## 搭建集群
## 部署管理平面
### federation 命名空间
```shell
kubectl --context="admin@kubernetes" create -f ns/federation.yaml
```
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: federation
```
### 联邦 API server
#### 部署联邦 apiserver Service
```shell
kubectl --context="admin@kubernetes" create -f services/federation-apiserver.yaml
```
需要修改，加入externalIPs，联邦控制平面对外暴露的访问IP，会被federation-controller-manager使用
```yaml
@@ -10,6 +10,8 @@ spec:
   selector:
     app: federated-cluster
     module: federation-apiserver
+  externalIPs:
+    - 10.213.128.80
   ports:
     - name: https
       protocol: TCP
```
#### 创建联邦apiserver Secret
修改 known-tokens.csv
```shell
kubectl --context="admin@kubernetes" --namespace=federation create secret generic federation-apiserver-secrets --from-file=known-tokens.csv
kubectl --context="admin@kubernetes" --namespace=federation describe secret federation-apiserver-secrets
```
#### 部署联邦 apiserver Deployment
需要配置联邦控制平面etcd的PV/PVC存储，可以emptyDir替代
修改 deployments/federation-apiserver.yaml
```yaml
@@ -15,7 +15,7 @@ spec:
     spec:
       containers:
       - name: apiserver
-        image: gcr.io/google_containers/hyperkube-amd64:v1.4.0
+        image: 10.213.42.254:10500/ming/hyperkube-amd64:v1.4.0
         command:
           - /hyperkube
           - federation-apiserver
@@ -23,7 +23,7 @@ spec:
           - --etcd-servers=http://localhost:2379
           - --service-cluster-ip-range=10.10.0.0/24
           - --secure-port=443
-          - --advertise-address=ADVERTISE_ADDRESS
+          - --advertise-address=10.213.128.80
           - --token-auth-file=/srv/kubernetes/known-tokens.csv
         ports:
           - containerPort: 443
@@ -35,7 +35,7 @@ spec:
             mountPath: /srv/kubernetes/
             readOnly: true
       - name: etcd
-        image: quay.io/coreos/etcd:v3.0.7
+        image: quay.io/coreos/etcd:v3.0.9
         command:
           - "/usr/local/bin/etcd"
         args:
@@ -48,5 +48,4 @@ spec:
           secret:
             secretName: federation-apiserver-secrets
         - name: etcd-data
-          persistentVolumeClaim:
-            claimName: federation-apiserver-etcd
+          emptyDir: {}
```
```shell
kubectl --context="admin1@mycluster1" --namespace=federation create -f deployments/federation-apiserver.yaml
kubectl --context="admin1@mycluster1" --namespace=federation get deploy
kubectl --context="admin1@mycluster1" --namespace=federation get po
```

### 联邦 Controller Manager
#### 修改deployment/federation-controller-manager.yaml
```yaml
@@ -19,12 +19,12 @@ spec:
           path: /etc/ssl
       containers:
       - name: controller-manager
-        image: gcr.io/google_containers/hyperkube-amd64:v1.4.0
+        image: 10.213.42.254:10500/ming/hyperkube-amd64:v1.4.0
         args:
           - /hyperkube
           - federation-controller-manager
           - --master=https://federation-apiserver:443
-          - --dns-provider=google-clouddns
```
#### 创建联邦apiserver Kubeconfig
联邦CM需要kubeconfig文件连接联邦apiserver
```shell
FEDERATED_API_SERVER_ADDRESS=10.213.128.80  // 联邦apiserver公共访问IP
kubectl config set-cluster federation-cluster --server=https://${FEDERATED_API_SERVER_ADDRESS} --insecure-skip-tls-verify=true
FEDERATION_CLUSTER_TOKEN=$(cut -d"," -f1 known-tokens.csv)
kubectl config set-credentials federation-cluster --token=${FEDERATION_CLUSTER_TOKEN}
kubectl config set-context federation-cluster --cluster=federation-cluster --user=federation-cluster
kubectl config use-context federation-cluster
kubectl config view --flatten --minify > kubeconfig/federation-apiserver/kubeconfig  // 导出联邦apiserver的凭据
// 创建联邦apiserver的Secret，需切换回宿主集群的context
kubectl --context="admin1@mycluster1" --namespace=federation create secret generic federation-apiserver-kubeconfig --from-file=kubeconfigs/federation-apiserver/kubeconfig
kubectl --context="admin1@mycluster1" --namespace=federation describe secret federation-apiserver-kubeconfig -
// 部署联邦 CM
kubectl --context="admin1@mycluster1" --namespace=federation create -f deployments/federation-controller-manager.yaml
kubectl --context="admin1@mycluster1" --namespace=federation get po
```

### 添加集群
添加集群需要：
1. 每个集群有效的kubeconfig，保存在secret
2. 在联邦apiserver里添加 cluster对象

#### 导出集群的kubeconfig （以添加宿主集群为例）
```shell
kubectl config use-context admin1@mycluster1 // 宿主集群
kubectl config view --flatten --minify > kubeconfig/k8s/kubeconfig  // 导出宿主集群的凭据
```

#### 创建相应 secret
```shell
kubectl --context="admin@kubernetes" --namespace=federation create secret generic mycluster1 --from-file=kubeconfigs/k8s/kubeconfig
```

#### 添加cluster对象
创建 k8s.yaml
```yaml
apiVersion: federation/v1beta1
kind: Cluster
metadata:
  name: mycluster1
spec:
  serverAddressByClientCIDRs:
    - clientCIDR: "0.0.0.0/0"
      serverAddress: "https://10.213.128.76:6443"
  secretRef:
    name: mycluster1
```
```shell
kubectl --context=federation-cluster create -f clusters/k8s.yaml
// 验证
kubectl --context=federation-cluster get clusters
```

### 更新kubeDNS

### 验证
部署Nginx，4个副本，默认平均分配到集群
```shell
kubectl --context=federation-cluster create -f rs/nginx.yaml
kubectl --context=federation-cluster get rs
kubectl --context=admin1@mycluster1 get rs
kubectl --context=admin2@mycluster2 get rs
```
不同集群的权重分配
```shell
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: nginx-us
  annotations:
    federation.kubernetes.io/replica-set-preferences: |
        {
            "rebalance": true,
            "clusters": {
                "mycluster1": {
                    "minReplicas": 2,
                    "maxReplicas": 4,
                    "weight": 1
                },
                "mycluster2": {
                    "minReplicas": 0,
                    "maxReplicas": 1,
                    "weight": 1
                }
            }
        }
```






