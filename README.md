# kubeadm-with-lxc


Assuming you're using a Ubuntu 20.04 LTS as the host mahcine to host your LXC containers:


1. Verify LXC version:

`lxc version`

```
 Client version: 4.0.3
 Server version: 4.0.3
```

2. Add a profile like below:

```
config:
  limits.cpu: "2"
  limits.memory: 2GB
  limits.memory.swap: "false"
  linux.kernel_modules: ip_tables,ip6_tables,netlink_diag,nf_nat,overlay
  raw.lxc: "lxc.apparmor.profile=unconfined\nlxc.cap.drop= \nlxc.cgroup.devices.allow=a\nlxc.mount.auto=proc:rw sys:rw"
  security.nesting: "true"
  security.privileged: "true"
description: Default LXD profile for k8s
devices:
  eth0:
    name: eth0
    network: lxdbr0
    type: nic
  root:
    path: /
    pool: default
    type: disk
name: k8s
```

3. Verify profile contents:
 
 `lxc profile show k8s`

4. Create two containers for kuebadm setup - master and worker 

`lxc launch ubuntu:18.04 kubeadm-master --profile k8s`
`lxc launch ubuntu:18.04 kubeadm-worker --profile k8s`

5. Exec to master:

`lxc exec kubeadm-master bash`

  5.1. Run the next commands on the kubeadm-master container

```
    1  sudo apt-get update
    2  cat <<EOF > /etc/sysctl.d/k8s.conf
       net.bridge.bridge-nf-call-ip6tables = 1
       net.bridge.bridge-nf-call-iptables = 1
       EOF
    3  sudo sysctl --system
    4  sudo apt-get install -y iptables arptables ebtables
    5  sudo apt-get install docker.io
    6  sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    7  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    8  cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
       deb https://apt.kubernetes.io/ kubernetes-xenial main
       EOF
    9  sudo apt-get update
   10  sudo apt-get install -y kubelet kubeadm kubectl
```   
   Below command reqd. for kubelet to run properly
``` 
   11  sudo ln -s /dev/console /dev/kmsg
```
   Get eth0 address to use for --apiserver-advertise-address param below:
```   
   12  ifconfig
```   
   
```
   13  kubeadm init --apiserver-advertise-address=10.204.14.8 --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=all
```   
   
   Make note of the kubeadm join command after the above command successfuly completes.Looks something like this:

```   
   kubeadm join 10.204.14.8:6443 --token xcjw5r.1vft727wrqpanvxn \
    --discovery-token-ca-cert-hash sha256:55a75587a23eaa641edcc9966d2b8eb9b05e5b0f526178c90b15358a10f402d1
```
   
   Setup kubeconfig:
```   
   14 mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```   
   14  kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
   15  kubectl get po -n kube-system
   16  kubectl get nodes
   17  kubeadm token create --print-join-command
```
6. Exec to worker from a diff. terminal

`lxc exec kubeadm-worker bash`

  6.1 Run next commands on the kubeadm-worker container

```
    1  sudo apt-get update
    2  cat <<EOF > /etc/sysctl.d/k8s.conf
       net.bridge.bridge-nf-call-ip6tables = 1
       net.bridge.bridge-nf-call-iptables = 1
       EOF
    3  sudo sysctl --system
    4  sudo apt-get install -y iptables arptables ebtables
    5  sudo apt-get install docker.io
    6  sudo apt-get update && sudo apt-get install -y apt-transport-https curl
    7  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    8  cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
       deb https://apt.kubernetes.io/ kubernetes-xenial main
       EOF
    9  sudo apt-get update
   10  sudo apt-get install -y kubelet kubeadm kubectl
   11  sudo ln -s /dev/console /dev/kmsg
```   
   Run kubeadm join command copied from master:

```
   kubeadm join 10.204.14.8:6443 --token xcjw5r.1vft727wrqpanvxn \
    --discovery-token-ca-cert-hash sha256:55a75587a23eaa641edcc9966d2b8eb9b05e5b0f526178c90b15358a10f402d1
```
 7. Once the join command succeeds, go back to masteer terminal and verfiy node has joined:
 
 `kubeclt get nodes -o wide`

