# Devops D3.4

DevOps part D3.4 (Kubernetes, Minikube, Secrets, Nginx)

## Описание:

Итоговое задание модуля базируется на предыдущих заданиях в юнитах.

Вам предлагается:
* создать сущность Deployment c тремя репликами веб-сервера;
* добавить к нему сервис;
* интегрировать его внутрь конфигурации с помощью ConfigMap;
* добавить секрет для создания basic auth аутентификации в nginx.

## Задание:

1) Создать Deployment со свойствами ниже:
* образ — nginx:1.21.1-alpine;
* имя — nginx-sf;
* количество реплик — 3.
2) Создать конфигурационный файл для нашего приложения и поместить его в наш Pod со следующими свойствами:
* путь до файла в Pod’е — /etc/nginx/nginx.conf;
* содержимое файла:
```code
user nginx;
worker_processes  1;
events {
  worker_connections  10240;
}
http {
  server {
      listen       80;
      server_name  localhost;
      location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
  }
}
```
3) Создать service для того, чтобы можно было обращаться к любому из Pod’ов по единому имени:
* имя сервиса sf-webserver;
* внешний порт — 80.
4) Создать секрет со следующими данными:
* имя секрета — auth_basic;
* ключ объекта в секрете — user1;
* значение объекта в секрете user1 — password1;
5) Подключить в наш контейнер эти секреты.
* Обновить конфиг nginx таким образом, чтобы подключенные секреты использовались для авторизации для доступа к странице по умолчанию в nginx.

## Полезные ссылки:

- Использование секретов в Pod’е как переменную окружения; (https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables)

## Реализация

Если Docker настроен так, что запускается только от sudo необходимо предварительно дать доступ для запуска от текущего пользователя:
```console
sudo usermod -aG docker $USER
sudo setfacl -m user:$USER:rw /var/run/docker.sock
```

## Подготовка в K8S

1) Устанвока пакета apache2-utils
```console
sudo apt install apache2-utils
```
2) Создаём пользователя user1 с паролем password1
```console
htpasswd -c auth user1
New password:
Re-type new password: 
Adding password for user user1
```

3) Смотрим содержимое и вносим его в секрет:
```console
cat auth
user1:$apr1$oGsCwaJq$lmn4ZNfHh5HvwaRJKLbFf1
```
Add to secret.yaml:
```console
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: auth_basic
stringData:
  user1: user1:$apr1$oGsCwaJq$lmn4ZNfHh5HvwaRJKLbFf1
```
Add to nginx-deployment.yaml:
```console
...
        volumeMounts:
        - name: userpass
          mountPath: "/etc/apache2/.htpasswd"
          readOnly: true
      volumes:
        - name: userpass
          secret:
            secretName: auth_basic
            optional: false  
```
Add to nginx.conf:
```console
...
          location / {
            auth_basic           "Top Secvret Area!";
            auth_basic_user_file /etc/nginx/htpasswd/user1;           
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
```

## Запуск в K8S

1) Install minikube. To install the latest minikube stable release on x86-64 Linux using Debian package:
```console
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```
2) Start minikube
```console
minikube start
```
3) Enable Ingress Addon
```console
minikube addons enable ingress
```
4) Goto folder with YAML file and Apply Kubectl manifest
```console
cd /D3.4
kubectl apply -f .
```
5) Get ingress info for show IP
```console
kubectl get ingress
NAME              CLASS   HOSTS              ADDRESS        PORTS   AGE
sf-nginx          nginx   sf-nginx.info      192.168.49.2   80      2m3s
```
6) Add to host
```console
sudo nano /etc/hosts
192.168.49.2   sf-nginx.info
```
7) Open in browser
```console
http://sf-nginx.info/
```
8) For show all objects
```console
kubectl get ingress,deployment,pod,svc -o wide    # current namespace
kubectl get ingress,deployment,pod,svc -A -o wide # all namespace
```
9) For show decribe by object
```console
kubectl describe svc sf-webserver
```
10) For Show Dashboard
```console
minikube dashboard
```