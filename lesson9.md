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
##### 3. Старт minikube с драйвером docker и проверка установки
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
##### 4. Создаем свой namespace
```
kubectl create namespace test
```
```
kubectl get pods -n test
```
#### Общий обзор
Во время установки minikube происходит конфигурирование kubernetes.
Kubectl получает информацию о конфигурации minikube из файлика config в каталоге ~/.kube
Там указываются параметры подключения, сертификаты и т.п.

### Установка конейнера с postgres
#### Манифест установки
> https://kubernetes.io/docs/concepts/workloads/pods/
##### Создаем файлик манифеста postgres.yaml
Создаем в любой директории, напр в домашней, откуда  откуда будем его непосредственно использовать при запуске.

apiVersion: v1
kind: Pod
metadata:
  name: maindb
  labels:
	  app.kubernetes.io/name: maindb
spec:
  containers:
  - name: maindb
    image: postgres:14
    ports:
    - containerPort: 5432
	  name: maindb-port
    env:
    - name: POSTGRES_PASSWORD
      value: postgres

##### Пускаем

kubectl apply -f postgres.yaml

Файлик postgres.yaml при этом закидывается в кубер в дефолтный namespace.

##### Проверяем

kubectl get pods

Контейнер postgres на месте.

##### Информация о kubectl и pod

kubectl get events

kubectl describe pods maindb

Видим State, POSTGRES_PASSWORD и т.д.

##### Смотрим логи

kubectl logs maindb

##### Заходим внутрь контейнера

kubectl exec -it maindb bash

su postgres
cd
psql

Postgres развернут.

#### Пытаемся прокинуть наружу порт
> https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/
##### Прокидывание
kubectl expose pod maindb --type=NodePort --port=5432
##### Прописываем сервис в postgres.yaml для возможности работы контейнера наружу

apiVersion: v1
kind: Pod
metadata:
  name: maindb
  labels:
	app.kubernetes.io/name: maindb
spec:
  containers:
  - name: maindb
    image: postgres:14
    ports:
    - containerPort: 5432
	  name: maindb-port
    env:
    - name: POSTGRES_PASSWORD
      value: postgres

---

apiVersion: v1
kind: Service
metadata:
  name: maindb
spec:
  selector:
    app.kubernetes.io/name: maindb
  type: NodePort
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 5432
    targetPort: maindb-port
  nodePort: 30007

##### Рестартуем pod и смотрим сервис

kubectl delete pods maindb

kubectl apply -f postgres.yaml

kubectl get service

#### Ставим postgres снаружи (для psql)
Можно поставить просто клиентские тулзы

sudo apt-get -y install postgres

##### Смотрим параметры сервиса (IP, port)

kubectl get service

##### Пробуем зайти

psql -h <public IP> -p <get service port> -U postgres -W
psql -h <get service IP> -p <get service port> -U postgres -W  

##### Пробуем заставить прокинуть порт
  
minikube service maindb --url  
  
psql -h <--url IP> -p <--url port> -U postgres -W   
  
Зашли.  

Но по <public IP> не получается. Делаем вывод о том, что minikube в docker имеет ограничение для проброса наружу.
Во всяком случае в YCloud по <public IP> это сделать не удается.



















































