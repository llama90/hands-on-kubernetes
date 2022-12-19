# 설치

가용 가능한 데스크탑에 K8s를 설치하기 때문에 주어진 Vagrant의 리소스 및 네트워크 설정을 변경했다. 그리고 Flannel 설치를 위해 일부 설정을 변경했다.

## 쿠버네티스 클러스터 설치

Vagrant를 이용해 가상 머신에 쿠버네티스 클러스터를 구성한다. 클러스터 구성을 위한 Vagrantfile을 그대로 사용한것이 아니라 내 환경에 맞게 일부 변경했으며, 변경 사항은 아래와 같다.

* 데스크탑에 설치하기 때문에 `public` 네트워크로 변경
* [Flannel](https://github.com/flannel-io/flannel) 설치를 위해 일부 설치 과정 변경
    > Flannel is a simple and easy way to configure a layer 3 network fabric designed for Kubernetes.

```bash
$ vagrant up
```

약 10~15분 정도 기다리면 가상 머신으로 쿠버네티스 클러스터가 구성된 것을 확인할 수 있고, worker 노드를 master 노드에 등록해주는 작업을 수행한다.

```bash
[root@k8s-master ~]# cat ~/join.sh
kubeadm join 192.168.56.30:6443 --token bver73.wda72kx4afiuhspo --discovery-token-ca-cert-hash sha256:7205b3fd6030e47b74aa11451221ff3c77daa0305aad0bc4a2d3196e69eb42b7

[root@k8s-node1 ~]# kubeadm join 192.168.56.30:6443 --token bver73.wda72kx4afiuhspo --discovery-token-ca-cert-hash sha256:7205b3fd6030e47b74aa11451221ff3c77daa0305aad0bc4a2d3196e69eb42b7

[root@k8s-node2 ~]# kubeadm join 192.168.56.30:6443 --token bver73.wda72kx4afiuhspo --discovery-token-ca-cert-hash sha256:7205b3fd6030e47b74aa11451221ff3c77daa0305aad0bc4a2d3196e69eb42b7
```

## 클라어인트 설정

admin 인증서를 `$HOME/.kube/config` 경로에 복사하는 스크립트는 Vagrant를 통해 수행된다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

클러스터에 생성되어 있는 kubeconfig 파일을 클라이언트에 복사하여, `kubectl`를 통해 클러스터를 제어할 수 있도록 한다.

```bash
mkdir -p $HOME/.kube
scp -p {CLUSTER_USER_ID}@{CLUSTER_IP}:~/.kube/config ~/.kube/config
```

`kubectl` 명령어가 정상동작하는지 확인한다.

```bash
$ kubectl get nodes
NAME         STATUS   ROLES                  AGE    VERSION
k8s-master   Ready    control-plane,master   2d3h   v1.22.0
k8s-node1    Ready    <none>                 2d2h   v1.22.0
k8s-node2    Ready    <none>                 2d2h   v1.22.0
```

## nginx Pod 배포

간단한 애플리케이션을 배포하여 동작을 확인한다.

```bash
$ kubectl create -f nginx/nginx-pod.yaml
pod/nginx-pod created

$ kubectl get pods
NAME        READY   STATUS              RESTARTS   AGE
nginx-pod   0/1     ContainerCreating   0          10s

$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          26s

$ kubectl port-forward pod/nginx-pod 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80

# browser 를 이용해 `localhost:8080`에 접속하면 해당 pod 에 액세스할 수 있다.

Handling connection for 8080
Handling connection for 8080
```

## Troubleshooting

### scheduler Unhealthy

설치후 클러스터 상태를 확인하니 아래와 같이 상태가 나타났다.

```bash
$ kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
controller-manager   Healthy     ok
etcd-0               Healthy     {"health":"true","reason":""}
```

`/etc/kubernetes/manifests/` 아래의 `kube-controller-manager.yaml` 및 `kube-schduler.yaml`이 설정한 기본 포트가 0이기 때문에 발생하는 문제로 해당 포트 설정을 주석 처리한다.

```diff
-   - --port=0
+ # - --port=0
```

```bash
# vim kube-controller-manager.yaml
# vim kube-scheduler.yaml
# systemctl restart kubelet.service
# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true","reason":""}
controller-manager   Healthy   ok
```

# 참고자료

* [k8s v1.22버전 설치 (최신, 쉬운설치 버전)](https://kubetm.github.io/k8s/02-beginner/cluster-install-case6/)
* [4.3. Install Kubernetes - Kubeadm](https://mlops-for-all.github.io/docs/setup-kubernetes/kubernetes-with-kubeadm/)
* [自学k8s-kubeadm部署过程中遇到的dial tcp 127.0.0.1:10251: connect: connection refused错误 ](https://www.cnblogs.com/potato-chip/p/13973760.html)