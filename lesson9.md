Постгрес в minikube
------------------------

## Развертывание

Делаем виртуалку с ubuntu в YCloud.

Поверяем требования

> https://kubernetes.io/ru/docs/tasks/tools/install-minikube/

```
grep -E --color 'vmx|svm' /proc/cpuinfo
```
Требования не выполняются. Будем ставить minikube докеровским контейнером.

### Ставим docker

> https://docs.docker.com/engine/install/ubuntu/

#### Uninstall all conflicting packages
```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
#### Set up the repository

##### 1. Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```
sudo apt-get update
```
```
sudo apt-get install ca-certificates curl gnupg
```
##### 2. Add Docker’s official GPG key
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```
##### 3. Set up the repository
```
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```  
#### Install Docker Engine 
  
##### 1. Update the apt package index
```
sudo apt-get update
```
##### 2. Install Docker Engine, containerd, and Docker Compose
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
##### 3. Verify that the Docker Engine installation is successful by running the hello-world image
```
sudo docker run hello-world
```
##### 4. Разбираемся с правами
```
sudo groupadd docker
```
```
sudo usermod -aG docker $USER
```
```
newgrp docker
```
##### 5. Проверяем 
```
docker images
```
```
docker ps
```
### Kubernetes
#### Установка kubectl в Linux
> https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-kubectl-%D0%B2-linux

##### 1. Последняя версия двичного kubectl
```
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
```
##### 2. Права
```
chmod +x ./kubectl
```
##### 3. Перемещаем в директорию из переменной окружения PATH
```
sudo mv ./kubectl /usr/local/bin/kubectl
```
##### 4. Проверяем версию
```
kubectl version --client
```
#### Minikube
> https://kubernetes.io/ru/docs/tasks/tools/install-minikube/

##### 1. Установка Minikube с помощью прямой ссылки
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
```  
##### 2. Чтобы исполняемый файл Minikube был доступен из любой директории  
```
sudo mkdir -p /usr/local/bin/
```
```
sudo install minikube /usr/local/bin/
```
##### 3. Проверка установки
> https://kubernetes.io/ru/docs/setup/learning-environment/minikube/#%D1%83%D0%BA%D0%B0%D0%B7%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B4%D1%80%D0%B0%D0%B9%D0%B2%D0%B5%D1%80%D0%B0-%D0%B2%D0%B8%D1%80%D1%82%D1%83%D0%B0%D0%BB%D1%8C%D0%BD%D0%BE%D0%B9-%D0%BC%D0%B0%D1%88%D0%B8%D0%BD%D1%8B
```
minikube start --vm-driver=docker
```
```
minikube status
```
```
docker ps
```
#### 4. Создаем свой namespace
```
kubectl create namespace test
```
```
kubectl get pods -n test
```











