# Домашнее задание к занятию "12.4 Развертывание кластера на собственных серверах, лекция 2"
Новые проекты пошли стабильным потоком. Каждый проект требует себе несколько кластеров: под тесты и продуктив. Делать все руками — не вариант, поэтому стоит автоматизировать подготовку новых кластеров.

## Задание 1: Подготовить инвентарь kubespray
Новые тестовые кластеры требуют типичных простых настроек. Нужно подготовить инвентарь и проверить его работу. Требования к инвентарю:
* подготовка работы кластера из 5 нод: 1 мастер и 4 рабочие ноды:
Ноды поднимаю vagrant'ом
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  config.vm.provider :virtualbox do |vbox|  
    vbox.customize ["modifyvm", :id, "--memory", 2048] #<= 4096 equals 4GB total memory.
    vbox.customize ["modifyvm", :id, "--cpus", 1]
  end
  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://vagrantcloud.com/search.
  config.vm.define "master" do |master|
    master.vm.box = "bento/ubuntu-20.04"
    master.vm.network :private_network, ip: "192.168.3.10" 
    master.vm.hostname = "master"
    master.vm.provision "shell" do |s|
     s.inline ="sudo apt-get update && sudo apt-get install -y python3-pip && pip3 install ansible;"
               end
  end
  config.vm.define "worker1" do |worker1|
    worker1.vm.box = "bento/ubuntu-20.04"
    worker1.vm.network :private_network, ip: "192.168.3.21"
    worker1.vm.hostname = "worker1"
  end
  config.vm.define :worker2 do |worker2|
    worker2.vm.box = "bento/ubuntu-20.04"
    worker2.vm.network :private_network, ip: "192.168.3.22"
    worker2.vm.hostname = "worker2"
  end
  config.vm.define :worker3 do |worker3|
    worker3.vm.box = "bento/ubuntu-20.04"
    worker3.vm.network :private_network, ip: "192.168.3.23"
    worker3.vm.hostname = "worker3"
  end
  config.vm.define :worker4 do |worker4|
    worker4.vm.box = "bento/ubuntu-20.04"
    worker4.vm.network :private_network, ip: "192.168.3.24"
    worker4.vm.hostname = "worker4"
  end
end
```
Не стал автоматизировать копирование ssh ключей, возможно зря.
inventory.ini, с hosts.yml тоже запустилось
```yml
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
master ansible_host=192.168.3.10  ip=192.168.3.10 
worker1 ansible_host=192.168.3.21 ip=192.168.3.21
worker2 ansible_host=192.168.3.22 ip=192.168.3.22
worker3 ansible_host=192.168.3.23 ip=192.168.3.23
worker4 ansible_host=192.168.3.24 ip=192.168.3.24
# node6 ansible_host=95.54.0.17  # ip=10.3.0.6 etcd_member_name=etcd6

# ## configure a bastion host if your nodes are not directly reachable
# [bastion]
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube_control_plane]
master
# node2
# node3

[etcd]
master
# node2
# node3

[kube_node]
worker1
worker2
worker3
worker4
# node6

[calico_rr]

[k8s_cluster:children]
kube_control_plane
kube_node
calico_rr
```
* в качестве CRI — containerd;
```bash
vi inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```
```yml
container_manager: containerd
```
```bash
vi inventory/group_vars/all/containerd.yml
```
```yml
containerd_registries:
  "docker.io":
    - "https://registry-1.docker.io"
```
* запуск etcd производить на мастере.
```bash
vi inventory/group_vars/etcd.yml
```
```yml
etcd_deployment_type: host
```
```bash
ansible-playbook -u vagrant -i inventory/homecluster/inventory.ini cluster.yml -b
```
```bash
kubectl get pod -o=custom-columns=NODE:.spec.nodeName,NAME:.metadata.name --all-namespaces
```
```bash
NODE      NAME
worker1   calico-kube-controllers-684bcfdc59-9pgwl
worker3   calico-node-785r4
master    calico-node-86fhk
worker2   calico-node-jf7d9
worker4   calico-node-qpknb
worker1   calico-node-wlv9b
master    coredns-8474476ff8-9stq8
worker1   coredns-8474476ff8-cnfvm
worker1   dns-autoscaler-5ffdc7f89d-g8vlv
master    kube-apiserver-master
master    kube-controller-manager-master
master    kube-proxy-bzvrm
worker2   kube-proxy-ltgpq
worker1   kube-proxy-rvjdg
worker3   kube-proxy-splwh
worker4   kube-proxy-vmqxp
master    kube-scheduler-master
worker1   nginx-proxy-worker1
worker2   nginx-proxy-worker2
worker3   nginx-proxy-worker3
worker4   nginx-proxy-worker4
worker2   nodelocaldns-fzj5b
worker4   nodelocaldns-hhlwx
worker3   nodelocaldns-l6vsn
<none>    nodelocaldns-vl7kx
worker1   nodelocaldns-xvd9j
```
