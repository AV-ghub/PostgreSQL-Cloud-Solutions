Постгрес в minikube
------------------------

## Развертывание

Делаем виртуалку с ubuntu.
Ставим curl.

Поверяем требования

> https://kubernetes.io/ru/docs/tasks/tools/install-minikube/

```
grep -E --color 'vmx|svm' /proc/cpuinfo
```
Требования не выполняются. Будем ставить minikube докеровским контейнером.

> https://docs.docker.com/engine/install/ubuntu/

### Ставим docker

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


























