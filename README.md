# Домашнее задание к занятию «Обновление приложений»

### Цель задания

Выбрать и настроить стратегию обновления приложения.

### Чеклист готовности к домашнему заданию

1. Кластер K8s.

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Документация Updating a Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment).
2. [Статья про стратегии обновлений](https://habr.com/ru/companies/flant/articles/471620/).

-----

### Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.

### Задание 2. Обновить приложение

1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.
2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть доступно.
3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.
4. Откатиться после неудачного обновления.

----

# Решение

## 1 Задание

Думаю, что для условий наиболее подходит стратегия `Rolling Update`.

Это стратегия обновления по умолчанию и она не требовательна к ресурсам.

Поскольку есть ограничения по ресурсам, то необходимо использовать параметры.

```bash
maxSurge: 20%
maxUnavailable: 20%
```

maxSurge: 20% даст возможность дополнительно запустить 20% реплик с новой версией приложения сверх имеющегося количества. Например, если у меня 5 реплик приложения, то maxSurge: 20% даст возможность запустить еще одну реплику приложения с новой версией.

maxUnavailable: 20% даст возможность выключить 20% реплик со старой версией приложения во время обновления. Например, если у меня 5 реплик приложения, то во время обновления будет выключена одна реплика чтобы дать ресурсы реплике с новым приложением.

Но если учитывать то, что обновления мажорное и новые версии приложения не умеют работать со старыми, то можно поставить параметр maxUnavailable: 100%. Это позволит не удалять старые реплики до проверки новых. Старые реплики будут занимать ресурсы, но если обращений к этим репликам не будет, то значительного потребления ресурсов они не вызовут. После проверки новых реплик можно будет удалить старые.

Обновления лучше проводить во время, когда зафиксирован наибольший период простоя кластера.

Если ресурсов совсем нет, то можно применить стратегию обновления `Recreate`.
Это приведет к остановке старых подов, соответственно прекращению любых запросов к нему.
После остановки старых подов создадутся новые поды и полностью удалятся старые поды.

## 2 Задание
```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# nano deployment_1.19.yaml
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  namespace: default
  annotations:
    kubernetes.io/change-cause: "nginx 1.19"
spec:
  selector:
    matchLabels:
      app: nmt
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 3
  template:
    metadata:
      labels:
        app: nmt
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "1180"
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl apply -f deployment_1.19.yaml
deployment.apps/nginx-multitool created
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl get po -o wide -w

NAME                                             READY   STATUS    RESTARTS   AGE     IP             NODE   NOMINATED NODE   READINESS GATES
nginx-multitool-785b9cc66b-5f6hl                 2/2     Running   0          89s     10.1.141.150   vm     <none>           <none>
nginx-multitool-785b9cc66b-6b22b                 2/2     Running   0          89s     10.1.141.149   vm     <none>           <none>
nginx-multitool-785b9cc66b-jbjf2                 2/2     Running   0          89s     10.1.141.152   vm     <none>           <none>
nginx-multitool-785b9cc66b-wbjxw                 2/2     Running   0          90s     10.1.141.151   vm     <none>           <none>
nginx-multitool-785b9cc66b-wbwwg                 2/2     Running   0          89s     10.1.141.153   vm     <none>           <none>
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl describe deployments.apps nginx-multitool

Name:                   nginx-multitool
Namespace:              default
CreationTimestamp:      Thu, 09 Jul 2026 16:50:40 +0300
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
                        kubernetes.io/change-cause: nginx 1.19
Selector:               app=nmt
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  3 max unavailable, 2 max surge
Pod Template:
  Labels:  app=nmt
  Containers:
   nginx:
    Image:        nginx:1.19
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
   multitool:
    Image:      wbitt/network-multitool
    Port:       8080/TCP
    Host Port:  0/TCP
    Environment:
      HTTP_PORT:   1180
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-multitool-785b9cc66b (5/5 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  3m27s  deployment-controller  Scaled up replica set nginx-multitool-785b9cc66b from 0 to 5
```

Обновляемся до версии 1.20

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# nano deployment_1.20.yaml
```
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  namespace: default
  annotations:
    kubernetes.io/change-cause: "update to nginx 1.20"
spec:
  selector:
    matchLabels:
      app: nmt
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 3
  template:
    metadata:
      labels:
        app: nmt
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "1180"
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl apply -f deployment_1.20.yaml
deployment.apps/nginx-multitool configured
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl get po -o wide -w

NAME                                             READY   STATUS              RESTARTS   AGE     IP             NODE   NOMINATED NODE   READINESS GATES
nginx-multitool-5f6bf6554-6ggqf                  0/2     ContainerCreating   0          40s     <none>         vm     <none>           <none>
nginx-multitool-5f6bf6554-d8sv6                  0/2     ContainerCreating   0          52s     <none>         vm     <none>           <none>
nginx-multitool-5f6bf6554-nzsmt                  0/2     ContainerCreating   0          42s     <none>         vm     <none>           <none>
nginx-multitool-5f6bf6554-wv6dh                  0/2     ContainerCreating   0          40s     <none>         vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  0/2     ContainerCreating   0          52s     <none>         vm     <none>           <none>
nginx-multitool-785b9cc66b-jbjf2                 2/2     Running             0          7m11s   10.1.141.152   vm     <none>           <none>
nginx-multitool-785b9cc66b-wbwwg                 2/2     Running             0          7m11s   10.1.141.153   vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  0/2     ContainerCreating   0          54s     <none>         vm     <none>           <none>
cnginx-multitool-5f6bf6554-wv6dh                  0/2     ContainerCreating   0          60s     <none>         vm     <none>           <none>
nginx-multitool-5f6bf6554-d8sv6                  2/2     Running             0          73s     10.1.141.154   vm     <none>           <none>
nginx-multitool-785b9cc66b-jbjf2                 2/2     Terminating         0          7m39s   10.1.141.152   vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  2/2     Running             0          89s     10.1.141.155   vm     <none>           <none>
nginx-multitool-5f6bf6554-nzsmt                  0/2     ContainerCreating   0          80s     <none>         vm     <none>           <none>
nginx-multitool-785b9cc66b-wbwwg                 2/2     Terminating         0          8m1s    10.1.141.153   vm     <none>           <none>
nginx-multitool-785b9cc66b-jbjf2                 2/2     Terminating         0          8m4s    10.1.141.152   vm     <none>           <none>
nginx-multitool-5f6bf6554-wv6dh                  2/2     Running             0          93s     10.1.141.156   vm     <none>           <none>
nginx-multitool-785b9cc66b-jbjf2                 2/2     Terminating         0          8m14s   10.1.141.152   vm     <none>           <none>
nginx-multitool-5f6bf6554-6ggqf                  0/2     ContainerCreating   0          103s    <none>         vm     <none>           <none>
nginx-multitool-785b9cc66b-wbwwg                 2/2     Terminating         0          8m21s   10.1.141.153   vm     <none>           <none>
```
Постепенно запустилось 5 подов с новой версией `nginx`.
В процессе обновления сначала 2 старых пода продолжали работать, 3 пода выключились и 3 пода создались, после чего 2 старых пода выключились и вместо них запустились 2 новых пода.
Версия `nginx` обновилась до 1.20

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl describe deployments.apps nginx-multitool

Name:                   nginx-multitool
Namespace:              default
CreationTimestamp:      Thu, 09 Jul 2026 16:50:40 +0300
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
                        kubernetes.io/change-cause: update to nginx 1.20
Selector:               app=nmt
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  3 max unavailable, 2 max surge
Pod Template:
  Labels:  app=nmt
  Containers:
   nginx:
    Image:        nginx:1.20
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
   multitool:
    Image:      wbitt/network-multitool
    Port:       8080/TCP
    Host Port:  0/TCP
    Environment:
      HTTP_PORT:   1180
    Mounts:        <none>
  Volumes:         <none>
  Node-Selectors:  <none>
  Tolerations:     <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  nginx-multitool-785b9cc66b (0/0 replicas created)
NewReplicaSet:   nginx-multitool-5f6bf6554 (5/5 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  5m8s   deployment-controller  Scaled up replica set nginx-multitool-5f6bf6554 from 0 to 2
  Normal  ScalingReplicaSet  5m7s   deployment-controller  Scaled down replica set nginx-multitool-785b9cc66b from 5 to 2
  Normal  ScalingReplicaSet  5m7s   deployment-controller  Scaled up replica set nginx-multitool-5f6bf6554 from 2 to 5
  Normal  ScalingReplicaSet  3m50s  deployment-controller  Scaled down replica set nginx-multitool-785b9cc66b from 2 to 1
  Normal  ScalingReplicaSet  3m28s  deployment-controller  Scaled down replica set nginx-multitool-785b9cc66b from 1 to 0
```

History deployment:
```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# kubectl -n default rollout history deployment

deployment.apps/nginx-multitool 
REVISION  CHANGE-CAUSE
1         nginx 1.19
2         update to nginx 1.20
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# nano deployment_1.28.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
  namespace: default
  annotations:
    kubernetes.io/change-cause: "update to nginx 1.28"
spec:
  selector:
    matchLabels:
      app: nmt
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 3
  template:
    metadata:
      labels:
        app: nmt
    spec:
      containers:
      - name: nginx
        image: nginx:1.28
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        ports:
        - containerPort: 8080
        env:
          - name: HTTP_PORT
            value: "1180"
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl apply -f deployment_1.28.yaml
deployment.apps/nginx-multitool configured
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl get po -o wide -w

NAME                                             READY   STATUS              RESTARTS   AGE     IP             NODE   NOMINATED NODE   READINESS GATES
nginx-multitool-5f6bf6554-6ggqf                  2/2     Terminating         0          10m     10.1.141.158   vm     <none>           <none>
nginx-multitool-5f6bf6554-d8sv6                  2/2     Terminating         0          10m     10.1.141.154   vm     <none>           <none>
nginx-multitool-5f6bf6554-nzsmt                  2/2     Running             0          10m     10.1.141.157   vm     <none>           <none>
nginx-multitool-5f6bf6554-wv6dh                  2/2     Running             0          10m     10.1.141.156   vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  2/2     Terminating         0          10m     10.1.141.155   vm     <none>           <none>
nginx-multitool-fdcdb7778-c5rsc                  0/2     ContainerCreating   0          35s     <none>         vm     <none>           <none>
nginx-multitool-fdcdb7778-g2vcw                  0/2     Pending             0          32s     <none>         vm     <none>           <none>
nginx-multitool-fdcdb7778-j66f6                  0/2     ContainerCreating   0          32s     <none>         vm     <none>           <none>
nginx-multitool-fdcdb7778-jk69x                  0/2     ContainerCreating   0          33s     <none>         vm     <none>           <none>
nginx-multitool-fdcdb7778-w9b7d                  0/2     ContainerCreating   0          36s     <none>         vm     <none>           <none>
nginx-multitool-fdcdb7778-g2vcw                  0/2     ContainerCreating   0          33s     <none>         vm     <none>           <none>
nginx-multitool-5f6bf6554-6ggqf                  0/2     Completed           0          10m     10.1.141.158   vm     <none>           <none>
nginx-multitool-5f6bf6554-6ggqf                  0/2     Completed           0          10m     10.1.141.158   vm     <none>           <none>
nginx-multitool-5f6bf6554-6ggqf                  0/2     Completed           0          10m     10.1.141.158   vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  0/2     Completed           0          10m     10.1.141.155   vm     <none>           <none>
nginx-multitool-5f6bf6554-d8sv6                  0/2     Completed           0          10m     10.1.141.154   vm     <none>           <none>
nginx-multitool-5f6bf6554-d8sv6                  0/2     Completed           0          10m     10.1.141.154   vm     <none>           <none>
nginx-multitool-5f6bf6554-d8sv6                  0/2     Completed           0          10m     10.1.141.154   vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  0/2     Completed           0          10m     10.1.141.155   vm     <none>           <none>
nginx-multitool-5f6bf6554-x7bmf                  0/2     Completed           0          10m     10.1.141.155   vm     <none>           <none>
nginx-multitool-fdcdb7778-w9b7d                  0/2     ContainerCreating   0          46s     <none>         vm     <none>           <none>
```

Две старые реплики продолжают работать, но три новые не могут запуститься из-за отсутствия образа nginx 1.28. За счет двух старых реплик доступность приложения сохраняется

Откат к предыдущей версии:
```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl rollout undo deployment nginx-multitool
deployment.apps/nginx-multitool rolled back
```

```bash
root@vm:/home/bogatyrevam/Desktop/9-update-app# microk8s kubectl rollout history deployment

deployment.apps/nginx-multitool 
REVISION  CHANGE-CAUSE
1         nginx 1.19
3         update to nginx 1.28
4         update to nginx 1.20
```







