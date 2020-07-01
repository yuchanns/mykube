# Dev-Kaka
The development environment for `LNMP`

## Prepare
* Install nfs-kernel-server in my Debian system
  ```sh
  # special setting for Chinese users
  sed -i 's/security.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
  sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
  # install nfs-kernel-server
  sudo apt update && apt install nfs-kernel-server rpcbind
  # clone this repository
  git clone git@github.com:yuchanns/mykube.git
  # enter and make it shared
  cd mykube && sudo chown nobody $(pwd)
  echo "$(pwd) *(rw,sync,all_squash)" >> /etc/exports
  # apply config
  sudo /usr/sbin/exportfs -ra
  ```

* Install minikube and nfs-common

  ```sh
  # install minikube according to https://kubernetes.io/zh/docs/setup/learning-environment/minikube/
  # then ssh into it and install nfs-common
  sed -i 's/security.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
  sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
  sudo apt update && apt install nfs-common rpcbind
  ```

## Usage

the usage is  work in progress.

```sh
kubectl create -f kaka/config.yaml
```
