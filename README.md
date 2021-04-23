# chevvi_platform  
chevvi Platform repository  

## Домашняя работа 1  


### Задание 1  
Разберитесь почему все pod в namespace kube-system восстановилисьпосле удаления.  

### Ответ  
~~~~  
$kubectl get pods -n kube-system  
NAME                               READY   STATUS    RESTARTS   AGE  
coredns-66bff467f8-wr2v6           1/1     Running   0          21h  
etcd-minikube                      1/1     Running   0          21h  
kube-apiserver-minikube            1/1     Running   0          21h  
kube-controller-manager-minikube   1/1     Running   0          21h  
kube-proxy-fdz8j                   1/1     Running   0          21h  
kube-scheduler-minikube            1/1     Running   0          21h  
~~~~

В неймспейсе kube-system существует ReplicaSet - coredns-66bff467f8, у которой указано количество реплик.    
~~~~
$kubectl get rs -n kube-system
NAME                 DESIRED   CURRENT   READY   AGE  
coredns-66bff467f8   1         1         1       22h  

$kubectl describe rs/coredns-66bff467f8 -n kube-system | grep Replicas  
Replicas:       1 current / 1 desired  
~~~~  

И ReplicaSet rs/coredns-66bff467f8 связан (слинкован) с подом coredns-66bff467f8-wr2v6 через поле metadata.ownerReferences  
~~~~  
$kubectl get pod coredns-66bff467f8-wr2v6 -n kube-system -o yaml | grep -A5 ownerReferences  
....
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: coredns-66bff467f8
....
~~~~

rs/coredns-66bff467f8 в свою очередь принадлежит контроллеру Deployment coredns.
~~~~
kubectl get rs/coredns-66bff467f8 -n kube-system -o yaml | grep -A5 ownerReferences
...
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: coredns
...
~~~~

У деплоймента coredns указано количество реплик.
~~~~
kubectl describe deployments coredns -n kube-system | grep 'Replicas:'
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
~~~~

Поэтому под coredns-66bff467f8-wr2v6 восстанавливается после удаления.

Поды kube-apiserver, etcd, kube-controller-manager, kube-scheduler - являются статическими подами.
Kubelet демон управляет ими напрямую, без участия API сервера.
kublet наблюдает за каждым статическим модулем и перезапускает из если они упали.

kube-proxy это принадлежит DaemonSet kube-proxy.
~~~~
kubectl get pod kube-proxy-fdz8j  -n kube-system -o yaml |  grep -A5 ownerReferences
...
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: DaemonSet
    name: kube-proxy
...
~~~~
У DaemonSet kube-proxy количество реплик.
~~~~
kubectl get ds -n kube-system
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-proxy   1         1         1       1            1           kubernetes.io/os=linux   25h

$kubectl describe daemonset kube-proxy -n kube-system | grep Nodes
Desired Number of Nodes Scheduled: 1
Current Number of Nodes Scheduled: 1
Number of Nodes Scheduled with Up-to-date Pods: 1
Number of Nodes Scheduled with Available Pods: 1
Number of Nodes Misscheduled: 0
~~~~

### Задание 2
2.1 Написан Dockerfile и на основе него собран образ.  
Используется образ нжинкса, в создаваемый образ копиуется конфигурация нжинкса, и homework.html, для отдачи.  
Образ отправлен в hub.docker.com.  
2.2 Написан web-prod.yaml.  
Используя этот манифест, был поднят под web.  
2.3 В web-prod.yaml был добавлен init container и volumes.  
Под был запущен заново (предыдущий был удален).  
Проверена работа приложения, проброшен порт, открывается страница.
~~~~
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000 
~~~~
2.4 Был собран собственный образ hipster-frontend, и запушен в hub.docker.com.  
Предпринята попытка запуска пода frontend.
Под находится в статусе Error, т.к. для работы ему не хватает переменных, которые он ожидает.

2.5 Задание со Звездочкой 
Под находится в статусе Error, т.к. для работы ему не хватает переменных, которые он ожидает.
kubectl logs frontend и kubectl describe pod frontend показывают состояние пода и переменные которые ему необходимы.

Добавлен исправленный манифест frontend-pod-healthy.yaml, который позволяет запустить под frontend так, чтобы он был в состоянии Running.  

## Домашняя работа 2  
Эта дамашняя работа выполнялас в кластере kind. 

### Задание 1  
Был создан и пременен манифест frontend-replicaset.yaml. Использовался образ созданный в рамках предыдущей ДЗ.

Применение первой версии манифеста завершилось неудачей, т.к. не хватало селектора, в котором должно быть указано условие (матчинг лейбла) принадлежности пода данному ReplicaSet.

## Почему обновление ReplicaSet не повлекло обновление запущенных pod?

### Ответ
Изменения внесенные в ReplicaSet, и в частности, обновление версии образа применяются только к новым под и никак не отражается на уже запущеных (раотающих).  


### Задание 2  
Были собраны 2 обараза микросервиса paymentervice и написан валидный paymentservice-replicaset.yaml поднимающий 3 реплики из образа v0.0.1.

Был написан paymentservice-deployment.yaml и применен. (он также должен поднимать 3 реплики из образа v0.0.1.
Произведено обновление подов (обновлена версия образа на v0.0.2), дефолтная стратегия обновления RollingUpdate:
в ходе которой создаётся 1 новый под с новой версией и затем удаляет старый под и.т.д.
В итоге все новые поды подняты из новой версии (v0.0.2), и у нас есть 2 ReplicaSet, каждый управляет тремя подами с разными версиями образов.

Следующей командой был произведен откат, до версии образа v0.0.1.
~~~~
kubectl rollout undo deployment paymentservice --to-revision=1 | kubectl get rs -l app=paymentservice -w
~~~~

### Задание со Звездочкой ⭐
C использованием параметров maxSurgeиmax Unavailable
Были написаны 2 манифеста, деплоймента микросервиса paymentservice:
- аналог blue-green - paymentservice-deployment-bg.yaml
- аналог Rolling Update - paymentservice-deployment-reverse.yaml


### Задание 3  
Создан манифест frontend-deployment.yaml. поднимающий микросервис frontend из образа с тего v0.0.1, добавлена readinessProbe.
Затем был обновлен манифет: поменял версию образа на v0.0.2 и внесена ошибка в URL пробы.
Убедился что проверка не была пройдена, есть следующая ошибка:
~~~~
  Warning  Unhealthy  6s (x7 over 66s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
~~~~

### Задание со Звездочкой ⭐
Был написан манифест node-exporter-daemonset.yaml - DaemonSet для поднятия node-exporter, проброшен порт и проверена работа (доступность) поднятого сервиса.


### Задание со Звездочкой ⭐⭐
Для поднятия подов под управлением DaemonSet на мастер нодах, необходимо добавить в манифест tolerations
~~~~
      tolerations:
        - key: node-role.kubernetes.io/master
~~~~


## Домашняя работа 3  

### Задание 1  
task01 
Создан Service Account bob, ему выдана роль admin в рамках всего кластера.  
Создан Service Account dave без доступа к кластеру.  

### Задание 2  
task02
Создан Namespace prometheus.
Создан Service Account carol Namespace prometheus.
Всем Service Account в Namespace prometheus выдана возможность делать get, list, watch в отношении Pods всего кластера.


### Задание 3  
task03
Создан Namespace dev.
Создан Service Account jane в Namespace dev и выдана роль admin в рамках Namespace dev.
Создан Service Account ken в Namespace dev и выдана роль view в рамках Namespace dev.


## Домашняя работа 4  

Домашняя работа выполнялась в kind.
Был поднят под содержащий контейнер minio. Как StatefulSet. Манифест - minio-statefulset.yaml.
В итоге создались:
- под с Minio
- PVC
- PV
~~~
$kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
minio-0   1/1     Running   0          5m44s

$kubectl get pvc
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-minio-0   Bound    pvc-727bbd6f-ec5f-402d-931f-3c775cd96a30   10Gi       RWO            standard       74m

$kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-727bbd6f-ec5f-402d-931f-3c775cd96a30   10Gi       RWO            Delete           Bound    default/data-minio-0   standard                73m
~~~

Для того, чтобы StatefulSet был доступен изнутри кластера, был создан Headless Service. Манифест - minio-headless-service.yaml.

### Задание ⭐   
Секреты, а именно
- MINIO_ACCESS_KEY
- MINIO_SECRET_KEY
были помещены в Secret, написан манифест minio-secret.yaml.
Секреты были кодированы в base64 
~~~
echo -n "<<some_secret>>" | base64
~~~

Был поправлен (на самом деле создан отдельный) манифест - minio-statefulset-secret.yaml.
В который были внесены следующие изменения
~~~
      containers:
      - name: minio
        env:
          - name: MINIO_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: minio-secret
                key: MINIO_ACCESS_KEY
          - name: MINIO_SECRET_KEY
            valueFrom:
              secretKeyRef:
                name: minio-secret
                key: MINIO_SECRET_KEY
~~~

Для проверки работы minio, авторизации с новыми секретами, был проброшен порт
~~~
kubectl port-forward pods/minio-0 9000:9000
~~~
И сделана авторизация как чераз minio-client, так череp web.


## Домашняя работа 5  
### Задание 1 
Сетевое взаимодействие Pod, сервисы
Работа с тестовым веб-приложением

было сделано:
- Добавление проверок Pod
- Создание объекта Deployment
- Добавление сервисов в кластер ( ClusterIP )
- Включение режима балансировки IPVS

В результат были написаны манифесты:
- web-deploy.yaml
- web-svc-cip.yaml
Доступ к приложению извне кластера

было сделано:
- Установка MetalLB в Layer2-режиме
- Добавление сервиса LoadBalancer
- Установка Ingress-контроллера и прокси ingress-nginx
- Создание правил Ingress

В результат были написаны манифесты:
- metallb-config.yaml
- web-svc-lb.yaml
- nginx-lb.yaml
- web-svc-headless.yaml
- web-ingress.yaml.
Доступ к CoreDNS снаружи кластера

В результат были написаны манифесты: 
- coredns/coredns-svc-tcp-lb.yaml 
- coredns/coredns-svc-udp-lb.yaml. 
