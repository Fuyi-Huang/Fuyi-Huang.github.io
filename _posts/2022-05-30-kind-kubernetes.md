---
title: 用kind安装kubernetes集群过程记录
tags: 
    - linux 
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
mathjax: true
mathjax_autoNumber: true
---


### 创建HTTPS证书密钥

```

sudo kubectl create secret tls tls-secret --key cert/fy-huang.cn-new.key  --cert cert/fy-huang.cn-new.crt

```

### 创建Ingress

利用已经创建好的密钥

```shell

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml


kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

```

```yaml

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

  name: tls-example-ingress

spec:

  tls:

  - hosts:

      - www.fy-huang.cn

    secretName: tls-secret

  rules:

  - host: www.fy-huang.cn

    http:

      paths:

      - path: /go

        pathType: Prefix

        backend:

          service:

            name: website-backend-service

            port:

              number: 8080

  - host: www.fy-huang.cn

    http:

      paths:

      - pathType: Prefix

        path: "/ss"

        backend:

          service:

            name: shadowsocks-service

            port:

              number: 10080

  - host: www.fy-huang.cn
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: website-frontend-service
            port:
              number: 80

```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  namespace: website
```

### 创建单例持久化MySQL

```yaml

apiVersion: apps/v1

kind: Deployment

metadata:
  namespace: website
  name: mysql

spec:

  selector:

    matchLabels:

      app: mysql

  strategy:

    type: Recreate

  template:

    metadata:

      labels:

        app: mysql

    spec:

      containers:

      - image: mysql:8.0.28

        name: mysql

        env:

          # Use secret in real usage

        - name: MYSQL_ROOT_PASSWORD

          value: xiaoman

        ports:

        - containerPort: 3306

          name: mysql

        volumeMounts:

        - name: mysql-persistent-storage

          mountPath: /var/lib/mysql

      volumes:

      - name: mysql-persistent-storage

        persistentVolumeClaim:

          claimName: mysql-pv-claim

```

上面创建了一个集群内其他pod可以访问的数据库, 如果要访问需要在集群内创建一个MySQL client的POD去连接

```shell

## 连接mysql

sudo kubectl run -it --rm --image=mysql:8.0.28 --restart=Never mysql-client -- mysql -h mysql -pxiaoman

```

### 创建静态页面反向代理

```Dockerfile

FROM openresty/openresty:1.19.9.1-10-buster-fat

WORKDIR ./

EXPOSE 80

COPY ./frontend /website

COPY ./openresty/conf/website.conf /etc/conf.d/website.conf

COPY ./openresty/nginx.conf /usr/local/openresty/nginx/conf/nginx.conf

```

```yaml

apiVersion: apps/v1

kind: Deployment

metadata:

  name: website-frontend

spec:

  selector:

    matchLabels:

      app: website-frontend

  strategy:

    type: RollingUpdate

  template:

    metadata:

      labels:

        app: website-frontend

    spec:

      containers:

      - image: fuyihuang/website-frontend:v0.0.1

        name: website-frontend

        ports:

        - containerPort: 80

          name: frontend-port


---


kind: Service

apiVersion: v1

metadata:

  name: website-frontend-service

spec:

  selector:

    app: website-frontend

  ports:

  # Default port used by the image

  - port: 80

```

### 部署后端应用镜像

```yaml

apiVersion: apps/v1

kind: Deployment

metadata:

  name: website-backend

spec:

  selector:

    matchLabels:

      app: website-backend

  strategy:

    type: RollingUpdate

  template:

    metadata:

      labels:

        app: website-backend

    spec:

      containers:

      - image: fuyihuang/website-backend:v0.0.1

        name: website-backend

        ports:

        - containerPort: 8080

          name: backend-port

```

```yaml

kind: Service

apiVersion: v1

metadata:

  name: website-backend-service

spec:

  selector:

    app: website-backend

  ports:

  # Default port used by the image

  - port: 8080

```

### 部署shadowsocks

```yaml

apiVersion: apps/v1

kind: Deployment

metadata:

  name: shadowsocks

spec:

  selector:

    matchLabels:

      app: shadowsocks

  strategy:

    type: RollingUpdate

  template:

    metadata:

      labels:

        app: shadowsocks

    spec:

      containers:

      - image: fuyihuang/shadowsocks-libev:v0.0.1

        name: shadowsocks

        ports:

        - containerPort: 10080

          name: shadowsocks

```

```yaml

kind: Service

apiVersion: v1

metadata:

  name: shadowsocks-service

spec:

  selector:

    app: shadowsocks

  ports:

  # Default port used by the image

  - port: 10080

---

apiVersion: networking.k8s.io/v1

kind: Ingress

metadata:

  name: example-ingress

spec:

  tls:

  - hosts:

      - www.fy-huang.cn

    secretName: tls-secret

  rules:

  - host: www.fy-huang.cn

    http:

      paths:

      - pathType: Prefix

        path: "/ss"

        backend:

          service:

            name: shadowsocks-service

            port:

              number: 10080

```
