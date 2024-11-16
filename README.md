**Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod**

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.

2. После запуска увеличить количество реплик работающего приложения до 2.

3. Продемонстрировать количество подов до и после масштабирования.

4. Создать Service, который обеспечит доступ до реплик приложений из п.1.

5. Создать отдельный Pod с приложением multitool и убедиться с помощью curl, что из пода есть доступ до приложений из п.1.


**Решение 1**

Создадим deployment.yaml со следующим содержимым

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multitool
spec:
  replicas: 1
  selector:
    matchLabels:
      app: multitool
  template:
    metadata:
      labels:
        app: multitool
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool:latest
        ports:
        - containerPort: 8080
```

Видим что наш под с мультитул приложением валится в ошибку

![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/1.jpg)


Зайдем влоги и разберемся в чем проблема

![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/2.jpg)


Видим что в контейнере multitool еще один экзепляр nginx занимает порт 80, который уже используется в другом контейнере, так как на одном порту пытаеются запуститься сразу 2 веб сервера, то возникает ошибка что у нас ничего не работает.


Чтобы исправить ее добавим в deployment.yaml ENV со следующим значением 

```
        env:
        - name: HTTP_PORT
          value: "7080"
```


![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/3.jpg)


Затем в deployment.yaml изменим значение replicas на 2, тем самым увеличив количество реплик приложения до двух


![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/4.jpg)


Затем созаддим файлик deployment-svc.yaml с содержимым для создания сервиса для доступа к репликам приложений

```
apiVersion: v1
kind: Service
metadata:
  name: deployment-svc
spec:
  selector:
    app: multitool
  ports:
  - name: for-nginx
    port: 80
    targetPort: 80
  - name: for-multitool
    port: 7080
    targetPort: 7080
```

Создаем и проверяем вси ли работает


![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/5.jpg)


Затем создадим отдельный под для приложения multitool multitool-app.yaml

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: multitool
  name: multitool-app
  namespace: default
spec:
  containers:
  - name: multitool
    image: wbitt/network-multitool
    ports:
    - containerPort: 8080
    env:
      - name: HTTP_PORT
        value: "7080"
```

Видим что создалось 

![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/6.jpg)


Теперь изнури новозоданного пода multitool-app проверим доступ до раннее созданных приложений


![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/7.jpg)

![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/8.jpg)

![Image alt](https://github.com/mezhibo/kubernetes3/blob/1f9f3234b032aaa9ea8b29c75c56c53098f60842/IMG/9.jpg)




**Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий**

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.

2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.

3. Создать и запустить Service. Убедиться, что Init запустился.

4. Продемонстрировать состояние пода до и после запуска сервиса.



**Решение 2**


Создадим deployment-init.yaml 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx-app
  name: nginx-app
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
      initContainers:
      - name: init-nginx-svc
        image: busybox
        command: ['sh', '-c', 'until nslookup nginx-svc.default.svc.cluster.local; do echo waiting for nginx-svc; sleep 5; done;']

```

Убедимся что nginx не стартует 

![Image alt](https://github.com/mezhibo/kubernetes3/blob/0decf37313e90e320521a999786d218442f9f559/IMG/10.jpg)

Создадим Service nginx-svc.yaml и убедимся что init запустился

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - name: http-port
    port: 80
    protocol: TCP
    targetPort: 80
```

Запускаем сервис, и сравниваем состояние пода до запуска сервиса и после

![Image alt](https://github.com/mezhibo/kubernetes3/blob/0decf37313e90e320521a999786d218442f9f559/IMG/11.jpg)



