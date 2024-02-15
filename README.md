# Домашнее задание к занятию "`Сетевое взаимодействие в K8S. Часть 2`" - `Никулин Михаил Сергеевич`



---

## Задание 1. Создать Deployment приложений backend и frontend

Создаем namespace для нового задания:
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get ns
NAME              STATUS   AGE
kube-system       Active   3d
kube-public       Active   3d
kube-node-lease   Active   3d
default           Active   3d
dz4               Active   23h

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl create ns dz5
namespace/dz5 created

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get ns
NAME              STATUS   AGE
kube-system       Active   3d
kube-public       Active   3d
kube-node-lease   Active   3d
default           Active   3d
dz4               Active   23h
dz5               Active   3s
```
### 1. Создать Deployment приложения _frontend_ из образа nginx с количеством реплик 3 шт.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dz5
spec:
  replicas: 3
  selector:
    matchLabels:
      app: main-fe
  template:
    metadata:
      labels:
        app: main-fe
    spec:
      containers:
        - image: nginx:1.19.2
          name: nginx

---
apiVersion: v1
kind: Service
metadata:
  name: svc-frontend
  namespace: dz5
spec:
  ports:
    - name: main-fe
      port: 80
  selector:
    app: main-fe
```
### 2. Создать Deployment приложения _backend_ из образа multitool. 
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: dz5
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main-be
  template:
    metadata:
      labels:
        app: main-be
    spec:
      containers:
        - image: wbitt/network-multitool
          name: multitool

---
apiVersion: v1
kind: Service
metadata:
  name: svc-backend
  namespace: dz5
spec:
  ports:
    - name: main-be
      port: 80
  selector:
    app: main-be

```
### 3. Добавить Service, которые обеспечат доступ к обоим приложениям внутри кластера. 
Поднимаем frontend:
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get pods -n dz5 -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE          NOMINATED NODE   READINESS GATES
frontend-54b75869ff-25nhv   1/1     Running   0          25s   10.1.123.178   netology-01   <none>           <none>
frontend-54b75869ff-2q4wg   1/1     Running   0          25s   10.1.123.179   netology-01   <none>           <none>
frontend-54b75869ff-2zhq4   1/1     Running   0          25s   10.1.123.180   netology-01   <none>           <none>

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get deployment -n dz5 -o wide
NAME       READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES         SELECTOR
frontend   3/3     3            3           39s   nginx        nginx:1.19.2   app=main-fe

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get svc -n dz5 -o wide
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
svc-frontend   ClusterIP   10.152.183.202   <none>        80/TCP    47s   app=main-fe

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get ep -n dz5 -o wide
NAME           ENDPOINTS                                         AGE
svc-frontend   10.1.123.178:80,10.1.123.179:80,10.1.123.180:80   60s
```
Поднимаем backend:
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl apply -f backend.yaml 
deployment.apps/backend created
service/svc-backend created

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get pods -n dz5 -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
frontend-54b75869ff-25nhv   1/1     Running   0          2m51s   10.1.123.178   netology-01   <none>           <none>
frontend-54b75869ff-2q4wg   1/1     Running   0          2m51s   10.1.123.179   netology-01   <none>           <none>
frontend-54b75869ff-2zhq4   1/1     Running   0          2m51s   10.1.123.180   netology-01   <none>           <none>
backend-c87cbc5f4-wkp47     1/1     Running   0          33s     10.1.123.182   netology-01   <none>           <none>

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get deployment -n dz5 -o wide
NAME       READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                    SELECTOR
frontend   3/3     3            3           2m55s   nginx        nginx:1.19.2              app=main-fe
backend    1/1     1            1           37s     multitool    wbitt/network-multitool   app=main-be

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get svc -n dz5 -o wide
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
svc-frontend   ClusterIP   10.152.183.202   <none>        80/TCP    2m59s   app=main-fe
svc-backend    ClusterIP   10.152.183.199   <none>        80/TCP    41s     app=main-be

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get ep -n dz5 -o wide
NAME           ENDPOINTS                                         AGE
svc-frontend   10.1.123.178:80,10.1.123.179:80,10.1.123.180:80   3m2s
svc-backend    10.1.123.182:80                                   43s
nikulinn@nikulin:~/other/kuber_1-5/scr$ 
```
### 4. Продемонстрировать, что приложения видят друг друга с помощью Service.
Сначала с ПОДов frontend:
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl exec -n dz5 frontend-54b75869ff-25nhv -- curl svc-backend 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0WBITT Network MultiTool (with NGINX) - backend-c87cbc5f4-wkp47 - 10.1.123.182 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
100   140  100   140    0     0  14000      0 --:--:-- --:--:-- --:--:-- 15555

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl exec -n dz5 frontend-54b75869ff-2q4wg -- curl svc-backend 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0    WBITT Network MultiTool (with NGINX) - backend-c87cbc5f4-wkp47 - 10.1.123.182 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
100   140  100   140    0     0   8235      0 --:--:-- --:--:-- --:--:--  8235

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl exec -n dz5 frontend-54b75869ff-2zhq4 -- curl svc-backend 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   140  100   140    0     0  17500      0 --:WBITT Network MultiTool (with NGINX) - backend-c87cbc5f4-wkp47 - 10.1.123.182 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
--:-- --:--:-- --:--:-- 17500
```
Теперь обратно:
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl exec -n dz5 backend-c87cbc5f4-wkp47 -- curl svc-frontend
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
100   612  100   612    0     0   191k      0 --:--:-- --:--:-- --:--:--  298k
```
### 5. Предоставить манифесты Deployment и Service в решении, а также скриншоты или вывод команды п.4.

------

## Задание 2. Создать Ingress и обеспечить доступ к приложениям снаружи кластера

### 1. Включить Ingress-controller в MicroK8S.
```bash
debian@netology-01:~$ microk8s status
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
  disabled:
    cert-manager         # (core) Cloud native certificate management
    cis-hardening        # (core) Apply CIS K8s hardening
    community            # (core) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    ingress              # (core) Ingress controller for external access
    kube-ovn             # (core) An advanced network fabric for Kubernetes
    mayastor             # (core) OpenEBS MayaStor
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    minio                # (core) MinIO object storage
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    registry             # (core) Private image registry exposed on localhost:32000
    rook-ceph            # (core) Distributed Ceph storage using Rook
    storage              # (core) Alias to hostpath-storage add-on, deprecated
    
debian@netology-01:~$ microk8s enable ingress
Infer repository core for addon ingress
Enabling Ingress
ingressclass.networking.k8s.io/public created
ingressclass.networking.k8s.io/nginx created
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
configmap/nginx-ingress-tcp-microk8s-conf created
configmap/nginx-ingress-udp-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled
```
Проверяем:
```bash
debian@netology-01:~$ microk8s status
microk8s is running
high-availability: no
  datastore master nodes: 127.0.0.1:19001
  datastore standby nodes: none
addons:
  enabled:
    dns                  # (core) CoreDNS
    ha-cluster           # (core) Configure high availability on the current node
    helm                 # (core) Helm - the package manager for Kubernetes
    helm3                # (core) Helm 3 - the package manager for Kubernetes
    ingress              # (core) Ingress controller for external access
  disabled:
    cert-manager         # (core) Cloud native certificate management
    cis-hardening        # (core) Apply CIS K8s hardening
    community            # (core) The community addons repository
    dashboard            # (core) The Kubernetes dashboard
    gpu                  # (core) Automatic enablement of Nvidia CUDA
    host-access          # (core) Allow Pods connecting to Host services smoothly
    hostpath-storage     # (core) Storage class; allocates storage from host directory
    kube-ovn             # (core) An advanced network fabric for Kubernetes
    mayastor             # (core) OpenEBS MayaStor
    metallb              # (core) Loadbalancer for your Kubernetes cluster
    metrics-server       # (core) K8s Metrics Server for API access to service metrics
    minio                # (core) MinIO object storage
    observability        # (core) A lightweight observability stack for logs, traces and metrics
    prometheus           # (core) Prometheus operator for monitoring and logging
    rbac                 # (core) Role-Based Access Control for authorisation
    registry             # (core) Private image registry exposed on localhost:32000
    rook-ceph            # (core) Distributed Ceph storage using Rook
    storage              # (core) Alias to hostpath-storage add-on, deprecated
```
### 2. Создать Ingress, обеспечивающий доступ снаружи по IP-адресу кластера MicroK8S так, чтобы при запросе только по адресу открывался _frontend_ а при добавлении /api - _backend_.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-dz
  namespace: dz5
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host:
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: svc-frontend
                port:
                  name: main-fe
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: svc-backend
                port:
                  name: main-be
```
Разворачиваем манифест ingress:
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl apply -f ingress.yaml 
ingress.networking.k8s.io/ingress-dz created

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl get ingress -n dz5
NAME         CLASS    HOSTS   ADDRESS     PORTS   AGE
ingress-dz   public   *       127.0.0.1   80      21s

nikulinn@nikulin:~/other/kuber_1-5/scr$ kubectl describe ingress ingress-dz -n dz5
Name:             ingress-dz
Labels:           <none>
Namespace:        dz5
Address:          127.0.0.1
Ingress Class:    public
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           
              /      svc-frontend:main-fe (10.1.123.178:80,10.1.123.179:80,10.1.123.180:80)
              /api   svc-backend:main-be (10.1.123.182:80)
Annotations:  nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason  Age                From                      Message
  ----    ------  ----               ----                      -------
  Normal  Sync    18s (x2 over 35s)  nginx-ingress-controller  Scheduled for sync
```
### 3. Продемонстрировать доступ с помощью браузера или `curl` с локального компьютера.
```bash
nikulinn@nikulin:~/other/kuber_1-5/scr$ curl 158.160.111.162
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

nikulinn@nikulin:~/other/kuber_1-5/scr$ curl 158.160.111.162/api
WBITT Network MultiTool (with NGINX) - backend-c87cbc5f4-wkp47 - 10.1.123.182 - HTTP: 80 , HTTPS: 443 . (Formerly praqma/network-multitool)
```
![img_2.png](img%2Fimg_2.png)
![img_1.png](img%2Fimg_1.png)
### 4. Предоставить манифесты и скриншоты или вывод команды п.2.


------