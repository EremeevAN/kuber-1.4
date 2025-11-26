### Домашнее задание к занятию «Сетевое взаимодействие в Kubernetes»
### Задание 1: Настройка Service (ClusterIP и NodePort)
Создать Deployment с двумя контейнерами:
nginx (порт 80).
multitool (порт 8080).
Количество реплик: 3.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginxmultitool
  annotations:
    container1: nginx
    container2: multitools
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: web
              containerPort: 80
          livenessProbe:
            tcpSocket:
              port: 80
            initialDelaySeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 4
        - name: multitool
          image: praqma/network-multitool:alpine-extra
          resources:
          env:
            - name: HTTP_PORT
              value: "8080"
            - name: HTTPS_PORT
              value: "8443"
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: https
              containerPort: 8443
              protocol: TCP
          livenessProbe:
            tcpSocket:
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 3
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 15
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 2
```
Создать Service типа ClusterIP, который:
Открывает nginx на порту 9001.
Открывает multitool на порту 9002.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: one
spec:
  selector:
    app: nginx-multitool
  type: ClusterIP
  ports:
    - name: nginx
      port: 9001
      targetPort: 80
    - name: multitool
      port: 9002
      targetPort: 8080
```
Проверить доступность изнутри кластера:

![image]

Создать Service типа NodePort для доступа к nginx снаружи.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nodeport
spec:
  selector:
    app: nginx-multitool
  type: NodePort
  ports:
    - name: nginx
      port: 80
      nodePort: 30111
    - name: multitool
      port: 8080
      nodePort: 30112
```
Проверить доступ с локального компьютера:

![image]

### Задание 2: Настройка Ingress
Развернуть два Deployment:
frontend (образ nginx).
backend (образ wbitt/network-multitool).
frontend
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  annotations:
    container1: nginx
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: nginx
      type: frontend
  template:
    metadata:
      labels:
        app: nginx
        type: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - name: web
              containerPort: 80
```
backend
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
  annotations:
    container1: multitool
spec:
  replicas: 1
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
      app: multitool
      type: backend
  template:
    metadata:
      labels:
        app: multitool
        type: backend
    spec:
      containers:
        - name: multitool
          image: praqma/network-multitool:alpine-extra
          resources:
          env:
            - name: HTTP_PORT
              value: "80"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```
Создать Service для каждого приложения.
frontend
```yaml
apiVersion: v1
kind: Service
metadata:
  name: two-one
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - name: nginx
      port: 80
      nodePort: 30111
```
backend
```yaml
apiVersion: v1
kind: Service
metadata:
  name: two-two
spec:
  selector:
    app: multitool
  type: NodePort
  ports:
    - name: multitool
      port: 80
      nodePort: 30112
```
Создать Ingress, который:
1. Открывает frontend по пути /.
2. Открывает backend по пути /api
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
           service:
            name: two-one
            port:
             name: nginx
      - path: /api
        pathType: Prefix
        backend:
        service:
            name: two-two
            port:
             name: multitool
```
Проверить доступность:

![image]

![image]