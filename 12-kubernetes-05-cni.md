# Домашнее задание к занятию "12.5 Сетевые решения CNI"
Кубер запускал на основе предыдущего задания
## Задание 1: установить в кластер CNI плагин Calico
После старта кластера 
```bash
$ calicoctl get profile projectcalico-default-allow -o json
```
```yml
{
  "kind": "Profile",
  "apiVersion": "projectcalico.org/v3",
  "metadata": {
    "name": "projectcalico-default-allow",
    "resourceVersion": "0",
    "creationTimestamp": null
  },
  "spec": {
    "ingress": [
      {
        "action": "Allow",
        "source": {},
        "destination": {}
      }
    ],
    "egress": [
      {
        "action": "Allow",
        "source": {},
        "destination": {}
      }
    ]
  }
}
```
Для задачи взял пример из лекции
```bash
$ kubectl get po
NAME                       READY   STATUS    RESTARTS      AGE
backend-7b4877445f-5zffn   1/1     Running   0             3h31m
cache-6df6d7d7df-rrdql     1/1     Running   0             3h31m
frontend-7f74b5fd7-7q8lx   1/1     Running   0             3h31m
```
создаю deployment+service curl
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: curl
  name: curl
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: curl
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
        - image: praqma/network-multitool:alpine-extra
          imagePullPolicy: IfNotPresent
          name: network-multitool
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: curl
  namespace: default
spec:
  ports:
    - name: web
      port: 80
  selector:
    app: curl
``` 
с него и будем стучаться
запрещаю все
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```
Открываю только для curl и только frontend
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: curl
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            app: curl
      ports:
        - protocol: TCP
          port: 80
        - protocol: TCP
          port: 443
```
```bash
$ kubectl exec curl-6c6f6b757-qrcsz -- curl -s backend -m 5
command terminated with exit code 28                                          
$ kubectl exec curl-6c6f6b757-qrcsz -- curl -s cache -m 5
command terminated with exit code 28                                          
$ kubectl exec curl-6c6f6b757-qrcsz -- curl -s frontend                                           
Praqma Network MultiTool (with NGINX) - frontend-7f74b5fd7-7q8lx - 10.233.110.6     
```
## Задание 2: изучить, что запущено по умолчанию
Самый простой способ — проверить командой calicoctl get <type>. Для проверки стоит получить список нод, ipPool и profile.
Требования: 
```bash
$ calicoctl get nodes
NAME      
master    
worker1   
worker2   
```
```bash
$ calicoctl get ipPool
NAME           CIDR             SELECTOR   
default-pool   10.233.64.0/18   all()      
```
```bash
$ calicoctl get profile
NAME                                                 
projectcalico-default-allow                          
kns.default                                          
kns.kube-node-lease                                  
kns.kube-public                                      
kns.kube-system                                      
kns.kubernetes-dashboard                             
ksa.default.default                                  
ksa.kube-node-lease.default                          
ksa.kube-public.default                              
ksa.kube-system.attachdetach-controller              
ksa.kube-system.bootstrap-signer                     
ksa.kube-system.calico-kube-controllers              
ksa.kube-system.calico-node                          
ksa.kube-system.certificate-controller               
ksa.kube-system.clusterrole-aggregation-controller   
ksa.kube-system.coredns                              
ksa.kube-system.cronjob-controller                   
ksa.kube-system.daemon-set-controller                
ksa.kube-system.default                              
ksa.kube-system.deployment-controller                
ksa.kube-system.disruption-controller                
ksa.kube-system.dns-autoscaler                       
ksa.kube-system.endpoint-controller                  
ksa.kube-system.endpointslice-controller             
ksa.kube-system.endpointslicemirroring-controller    
ksa.kube-system.ephemeral-volume-controller          
ksa.kube-system.expand-controller                    
ksa.kube-system.generic-garbage-collector            
ksa.kube-system.horizontal-pod-autoscaler            
ksa.kube-system.job-controller                       
ksa.kube-system.kube-proxy                           
ksa.kube-system.namespace-controller                 
ksa.kube-system.node-controller                      
ksa.kube-system.nodelocaldns                         
ksa.kube-system.persistent-volume-binder             
ksa.kube-system.pod-garbage-collector                
ksa.kube-system.pv-protection-controller             
ksa.kube-system.pvc-protection-controller            
ksa.kube-system.replicaset-controller                
ksa.kube-system.replication-controller               
ksa.kube-system.resourcequota-controller             
ksa.kube-system.root-ca-cert-publisher               
ksa.kube-system.service-account-controller           
ksa.kube-system.service-controller                   
ksa.kube-system.statefulset-controller               
ksa.kube-system.token-cleaner                        
ksa.kube-system.ttl-after-finished-controller        
ksa.kube-system.ttl-controller                       
ksa.kubernetes-dashboard.default                     
ksa.kubernetes-dashboard.kubernetes-dashboard
```
