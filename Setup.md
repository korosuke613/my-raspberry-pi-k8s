## 参考
https://qiita.com/kentarok/items/6e818c2e6cf66c55f19a

## OSインストール

### OSのダウンロード
2020-05-27-raspios-buster-lite-armhf.zip

### インストーラーのダウンロード
Raspberry Pi Imager

### SDカードに書き込み
Raspberry Pi Imagerを使う

### SSHを許可

```
cd /Volumes/boot
touch ssh
```

### 設定の追加
```
vim cmdline.txt
```

`cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`を追記

```
-> % cat cmdline.txt
console=serial0,115200 console=tty1 root=PARTUUID=2fed7fee-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait quiet init=/usr/lib/raspi-config/init_resize.sh cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

## IPアドレスの固定
sdカードをラズパイに入れて、lanケーブルをつないで通電。
ちょっと待つ。

```
arp -a
www.huaweimobilewifi.com (192.168.8.1) at <MAC_ADDRESS> on en0 ifscope [ethernet]
? (192.168.8.100) at <MAC_ADDRESS> on en0 ifscope [ethernet]
? (192.168.8.101) at <MAC_ADDRESS> on en0 ifscope [ethernet]
? (192.168.8.107) at <MAC_ADDRESS> on en0 ifscope [ethernet]
? (192.168.8.111) at <MAC_ADDRESS> on en0 ifscope [ethernet]
? (192.168.8.255) at ff:ff:ff:ff:ff:ff on en0 ifscope [ethernet]
? (224.0.0.251) at <MAC_ADDRESS> on en0 ifscope permanent [ethernet]
? (239.255.255.250) at <MAC_ADDRESS> on en0 ifscope permanent [ethernet]
broadcasthost (255.255.255.255) at ff:ff:ff:ff:ff:ff on en0 ifscope [ethernet]
```

初期パスワードは`raspberry`。あとで変える。
```
ssh pi@192.168.8.111
sudo nano /etc/dhcpcd.conf
```

```
# Example static IP configuration:
interface eth0
static ip_address=192.168.8.120/24
#static ip6_address=fd51:42f8:caae:d92e::ff/64
static routers=192.168.8.1
static domain_name_servers=192.168.8.1 8.8.8.8 fd51:42f8:caae:d92e::1
```

- Master Node
 - IP: 192.168.8.120
 - Host: k8s-master
- Worker Node 1
 - IP: 192.168.8.121
 - Host: k8s-node-1
- Worker Node 2
 - IP: 192.168.8.122
 - Host: k8s-node-2

## OSの更新
```
sudo apt-get update \
  && sudo apt-get -y dist-upgrade \
  && sudo apt-get -y autoremove \
  && sudo apt-get autoclean
```

## ホスト名変更
`k8s-master`の部分は適宜変更
```
sudo hostnamectl set-hostname k8s-master
sudo nano /etc/hosts
```

```
127.0.1.1		k8s-master
192.168.8.120		k8s-master
192.168.8.121		k8s-node-1
192.168.8.122		k8s-node-2
```

## Swap無効化
```
sudo dphys-swapfile swapoff
sudo systemctl stop dphys-swapfile
sudo systemctl disable dphys-swapfile
```

## SSH鍵でログイン

### SSH鍵の作成
1回のみ行う。
@mac
```
ssh-keygen -t rsa
```

`raspberry_rsa`という名前にした。パスフレーズは無し

### SSH鍵の登録
@mac
```
ssh-copy-id -i ~/.ssh/raspberry_rsa pi@192.168.8.120
```

### パスワード無しでログインできるか確認
@mac
```
ssh -i ~/.ssh/raspberry_rsa pi@192.168.8.120
```

### sshを楽にする
@mac
```
vim ~/.ssh/config
```

以下を追記
```
Host k8s-master
  HostName 192.168.8.120
  User pi
  IdentityFile ~/.ssh/raspberry_rsa
  Port 22
  TCPKeepAlive yes
  IdentitiesOnly yes
```

```
ssh k8s-master
```
これでsshが楽になった。

## ログインユーザ及びパスワード変更

### tmpユーザを作成
```
sudo useradd -M tmp
sudo gpasswd -a tmp sudo
sudo passwd tmp
```

### piの名前を変更
いったんexitしてsshし直す
tmp@k8s-master
```
sudo usermod -l <犬の名前> pi
```

### パスワード変更
新しいユーザ名でsshし直す
```
sudo passwd root
sudo passwd <新ユーザ名>
```

### tmpユーザの削除
```
sudo userdel tmp
```

## 電源に異常がないか確認
```
vcgencmd get_throttled
```

`throttled=0x0`なら大丈夫らしい

## docker導入
### インストール
```
curl https://get.docker.com | sh
```

```
$ docker -v
Docker version 19.03.11, build 42e35e6
```

### バージョン固定
```
sudo apt-mark hold docker-ce
```

### 実行権限の付与
```
sudo usermod -aG docker <ユーザ名>
```

## kubeadm, kubelet, kubectlのインストール
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

```
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:49:29Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/arm"}
```

```
$ kubelet --version
Kubernetes v1.18.3
```

```
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.3", GitCommit:"2e7996e3e2712684bc73f0dec0200d64eec7fe40", GitTreeState:"clean", BuildDate:"2020-05-20T12:52:00Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/arm"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

## Master Nodeの構築

### 要件を満たしているか確認
```
$ cat /proc/sys/net/bridge/bridge-nf-call-iptables
1
```

### kubeadmの設定
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

ログ
```
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16
W0614 11:40:50.569672   26846 version.go:102] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get https://dl.k8s.io/release/stable-1.txt: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
W0614 11:40:50.569847   26846 version.go:103] falling back to the local client version: v1.18.3
W0614 11:40:50.570162   26846 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.3
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.8.120]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.8.120 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.8.120 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0614 11:41:34.265069   26846 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0614 11:41:34.268456   26846 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 39.511314 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-check] Initial timeout of 40s passed.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: ynu9k9.q8hysemrtytkv97h
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.8.120:6443 --token ynu9k9.q8hysemrtytkv97h \
    --discovery-token-ca-cert-hash sha256:195f828fbb593aaeb777b5c646c4123ed11526811c0e901e6235b59de94de5f5
```

### kubectlの設定
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

確認
```
$ kubectl get node
NAME         STATUS     ROLES    AGE     VERSION
k8s-master   NotReady   master   2m18s   v1.18.3
```

#### bash補完設定
```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### Calicoのインストール

```
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
```

ログ
```
$ kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

### Worker Nodeを参加させる
@k8s-node-1
```
sudo kubeadm join 192.168.8.120:6443 --token ynu9k9.q8hysemrtytkv97h \
    --discovery-token-ca-cert-hash sha256:195f828fbb593aaeb777b5c646c4123ed11526811c0e901e6235b59de94de5f5
```

## できた？

```
$ kubectl version -o yaml
clientVersion:
  buildDate: "2020-05-20T12:52:00Z"
  compiler: gc
  gitCommit: 2e7996e3e2712684bc73f0dec0200d64eec7fe40
  gitTreeState: clean
  gitVersion: v1.18.3
  goVersion: go1.13.9
  major: "1"
  minor: "18"
  platform: linux/arm
serverVersion:
  buildDate: "2020-05-20T12:43:34Z"
  compiler: gc
  gitCommit: 2e7996e3e2712684bc73f0dec0200d64eec7fe40
  gitTreeState: clean
  gitVersion: v1.18.3
  goVersion: go1.13.9
  major: "1"
  minor: "18"
  platform: linux/arm
```

```
$ kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   13m   v1.18.3
k8s-node-1   Ready    <none>   2m    v1.18.3
```

```
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-66bff467f8-8qgjh             1/1     Running   0          13m
kube-system   coredns-66bff467f8-96bsg             1/1     Running   0          13m
kube-system   etcd-k8s-master                      1/1     Running   0          14m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          14m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          14m
kube-system   kube-flannel-ds-arm-nlwzb            1/1     Running   2          2m42s
kube-system   kube-flannel-ds-arm-s68ql            1/1     Running   0          4m7s
kube-system   kube-proxy-dgd4c                     1/1     Running   0          2m42s
kube-system   kube-proxy-f5qfv                     1/1     Running   0          13m
kube-system   kube-scheduler-k8s-master            1/1     Running   0          14m
```

できたっぽい！


## macからkubectlで操作

### configの確認
@k8s-master
```
kubectl config view --raw
```

上のコマンドで出てきた情報をいい感じでmacのconfigファイルに追加する。

@mac
```
vim ~/.kube/config
```

```
$ kubectl config get-contexts
CURRENT   NAME                 CLUSTER                      AUTHINFO             NAMESPACE
          docker-for-desktop   docker-for-desktop-cluster   docker-for-desktop
*         raspberry-k8s        kubernetes                   kubernetes-admin
$ kubectl config use-context raspberry-k8s
Switched to context "raspberry-k8s".
$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   39m   v1.18.3
k8s-node-1   Ready    <none>   28m   v1.18.3
```