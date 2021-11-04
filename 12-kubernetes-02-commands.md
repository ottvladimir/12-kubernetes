# Домашнее задание к занятию "12.2 Команды для работы с Kubernetes"

## Задание 1: Запуск пода из образа в деплойменте

```bash 
$ kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
$ kubectl edit deployments.apps
```
```yml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{"deployment.kubernetes.io/revision":"1"},"creationTimestamp":"2021-11-01T04:49:24Z","generation":1,"labels":{"app":"hello-node"},"name":"hello-node","namespace":"default","resourceVersion":"1056","uid":"94d86d09-d0df-4a4f-9f56-f72b28f83c49"},"spec":{"progressDeadlineSeconds":600,"replicas":1,"revisionHistoryLimit":10,"selector":{"matchLabels":{"app":"hello-node"}},"strategy":{"rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"},"type":"RollingUpdate"},"template":{"metadata":{"creationTimestamp":null,"labels":{"app":"hello-node"}},"spec":{"containers":[{"image":"k8s.gcr.io/echoserver:1.4","imagePullPolicy":"IfNotPresent","name":"echoserver","resources":{},"terminationMessagePath":"/dev/termination-log","terminationMessagePolicy":"File"}],"dnsPolicy":"ClusterFirst","restartPolicy":"Always","schedulerName":"default-scheduler","securityContext":{},"terminationGracePeriodSeconds":30}}},"status":{"availableReplicas":1,"conditions":[{"lastTransitionTime":"2021-11-01T04:49:57Z","lastUpdateTime":"2021-11-01T04:49:57Z","message":"Deployment has minimum availability.","reason":"MinimumReplicasAvailable","status":"True","type":"Available"},{"lastTransitionTime":"2021-11-01T04:49:24Z","lastUpdateTime":"2021-11-01T04:49:57Z","message":"ReplicaSet \"hello-node-7567d9fdc9\" has successfully progressed.","reason":"NewReplicaSetAvailable","status":"True","type":"Progressing"}],"observedGeneration":1,"readyReplicas":1,"replicas":2,"updatedReplicas":1}}
  creationTimestamp: "2021-11-01T04:49:24Z"
  generation: 5
  labels:
    app: hello-node
  name: hello-node
  namespace: default
  resourceVersion: "90582"
  uid: 94d86d09-d0df-4a4f-9f56-f72b28f83c49
spec:
  progressDeadlineSeconds: 600
  replicas: 2 # здесь заменил 1 на 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: hello-node
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-node
    spec:
      containers:
      - image: k8s.gcr.io/echoserver:1.4
        imagePullPolicy: IfNotPresent
        name: echoserver
        resources: {}
        terminationMessagePath: /dev/termination-log
```

## Задание 2: Просмотр логов для разработки

```
$ kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-node created
```
```
$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           53s
```
```
$ kubectl edit deployment/hello-node
```
Ставим replicas=2
```
deployment.apps/hello-node edited
```
```
$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   2/2     2            2           52m
```
```
kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-7567d9fdc9-kxpgs   1/1     Running   0          57m
hello-node-7567d9fdc9-x8g5v   1/1     Running   0          5m34s
```

## Задание 2: Просмотр логов для разработки
Создаю пользователя и сертификаты
```bash
# useradd minikube_pod_reader && cd /home/minikube_pod_reader/
$ openssl req -new -key minikube_pod_reader.key -out minikube_pod_reader.csr -subj "CN=minikube_pod_reader"
$ openssl x509 -req -in minikube_pod_reader.csr -CA /etc/pki/minikube/ca.crt -CAkey /etc/pki/minikube/ca.key -CAcreateserial -out minikube_pod_reader.crt -days 200
```
Создаю пользователя внутри Kubernetes
```bash
$ kubectl config set-credentials minikube_pod_reader --client-certificate=/home/minikube_pod_reader/.certs/minikube_pod_reader.crt --client-key=/home/minikube_pod_reader/.certs/minikube_pod_reader.key
```
Создаю контекст пользователя
```bash
$ kubectl config set-context minikube_pod_reader-context --cluster=minikube --user=minikube_pod_reader
```
Файл конфига для пользователя
```yml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /etc/pki/minikube/ca.crt
    extensions:
    - extension:
        last-update: Thu, 04 Nov 2021 17:49:01 +10
        provider: minikube.sigs.k8s.io
        version: v1.23.2
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Thu, 04 Nov 2021 17:49:01 +10
        provider: minikube.sigs.k8s.io
        version: v1.23.2
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/minikube_pod_reader/.certs/minikube_pod_reader.crt
    client-key: /home/minikube_pod_reader/.certs/minikube_pod_reader.key
```
Создаю Role и ClusterRole

```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: minikube:describe-and-logs
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log",]
  verbs: ["get", "list"]
```
```bash
$ kubectl apply -f role_log.yml
role.rbac.authorization.k8s.io/minikube:describe-and-logs created
```
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: minikube:describe-and-logs
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log",]
  verbs: ["get", "list"]
```
```bash
$ kubectl create -f clusterrole_log.yml
clusterrole.rbac.authorization.k8s.io/minikube:describe-and-logs created
```
Создаю RoleBinding
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: minikube_pod_reader
  namespace: default
subjects:
- kind: User
  name: minikube_pod_reader
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```
```bash
$ kubectl apply -f RoleBinding.yml
rolebinding.rbac.authorization.k8s.io/minikube_pod_reader created
```
Проверяю
```bash
$ su - minikube_pod_reader
$ kubectl logs -p hello-node-7567d9fdc9-x8g5v 
[minikube_pod_reader@mycentos ~]$ kubectl logs -p hello-node-7567d9fdc9-kxpgs 
[minikube_pod_reader@mycentos ~]$ kubectl describe pod hello-node-7567d9fdc9-x8g5v 
Name:         hello-node-7567d9fdc9-x8g5v
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Mon, 01 Nov 2021 15:41:29 +1000
Labels:       app=hello-node
              pod-template-hash=7567d9fdc9
Annotations:  <none>
Status:       Running
IP:           172.17.0.8
IPs:
  IP:           172.17.0.8
Controlled By:  ReplicaSet/hello-node-7567d9fdc9
Containers:
  echoserver:
    Container ID:   docker://06e8bb7a1cde83dc5bd4cf5ee208943f04d98aabbddfaf1f3d63d9090212bcab
    Image:          k8s.gcr.io/echoserver:1.4
    Image ID:       docker-pullable://k8s.gcr.io/echoserver@sha256:5d99aa1120524c801bc8c1a7077e8f5ec122ba16b6dda1a5d3826057f67b9bcb
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 04 Nov 2021 19:31:49 +1000
    Last State:     Terminated
      Reason:       Error
      Exit Code:    255
      Started:      Thu, 04 Nov 2021 17:49:02 +1000
      Finished:     Thu, 04 Nov 2021 19:31:33 +1000
    Ready:          True
    Restart Count:  2
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-h74j2 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-h74j2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason          Age   From     Message
  ----    ------          ----  ----     -------
  Normal  SandboxChanged  10m   kubelet  Pod sandbox changed, it will be killed and re-created.
  Normal  Pulled          10m   kubelet  Container image "k8s.gcr.io/echoserver:1.4" already present on machine
  Normal  Created         10m   kubelet  Created container echoserver
  Normal  Started         10m   kubelet  Started container echoserver
```
```bash
$ kubectl delete pod hello-node-7567d9fdc9-x8g5v 
Error from server (Forbidden): pods "hello-node-7567d9fdc9-x8g5v" is forbidden: User "minikube_pod_reader" cannot delete resource "pods" in API group "" in the namespace "default"
```

## Задание 3: Изменение количества реплик 

```bash
$ kubectl get deployments.apps 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   2/2     2            2           3d4h
$ kubectl scale --replicas=5 deployment hello-node 
$ kubectl get deployments.apps 
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   5/5     5            5           3d4h
```
