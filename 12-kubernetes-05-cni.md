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
* установить утилиту calicoctl;
* получить 3 вышеописанных типа в консоли.
