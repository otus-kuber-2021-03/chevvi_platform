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


## Домашняя работа 6  
Работа выполнялась в GCP
Для выполнения рыботы был установлен Helm 3. 
В процессе выполенения были:
- созданы неймспейсы
-- cert-manager
-- chartmuseum
-- harbor
-- hipster-shop
-- hipster-shop-prod
-- nginx-ingress
- созданы релизы
-- cert-manager
-- chartmuseum
-- harbor
-- hipster-shop
-- nginx-ingress
~~~~
kubectl get ns 
NAME                STATUS   AGE
cert-manager        Active   2d22h
chartmuseum         Active   2d21h
default             Active   2d23h
harbor              Active   30h
hipster-shop        Active   20h
hipster-shop-prod   Active   29m
kube-node-lease     Active   2d23h
kube-public         Active   2d23h
kube-system         Active   2d23h
nginx-ingress       Active   2d23h
helm ls -A
NAME         	NAMESPACE    	REVISION	UPDATED                                	STATUS  	CHART               	APP VERSION
cert-manager 	cert-manager 	1       	2021-05-01 19:36:39.961103429 +0300 MSK	deployed	cert-manager-v0.16.1	v0.16.1    
chartmuseum  	chartmuseum  	1       	2021-05-01 21:10:41.48334046 +0300 MSK 	deployed	chartmuseum-2.13.2  	0.12.0     
harbor       	harbor       	1       	2021-05-03 12:26:54.539744471 +0300 MSK	deployed	harbor-1.1.2        	1.8.2      
hipster-shop 	hipster-shop 	5       	2021-05-04 16:58:02.766401148 +0300 MSK	deployed	hipster-shop-0.1.0  	1.16.0     
nginx-ingress	nginx-ingress	1       	2021-05-01 19:27:35.189913923 +0300 MSK	deployed	nginx-ingress-1.41.3	v0.34.1   
Устанавливаю чарт хипстер-шопа  
~~~~
- выпущены сертификаты на необходимые домены (LE staging)

Был поправлен чарт хипстершопа, а именно вынесено несколько микросервисов в отдельные чарты и кастомизатор.
Далее приведены отдельные команды применявшиеся при выполнении работы.

~~~~
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop/ --namespace=hipster-shop
Release "hipster-shop" does not exist. Installing it now.
NAME: hipster-shop
LAST DEPLOYED: Mon May  3 21:56:25 2021
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 1
TEST SUITE: None
~~~~
Смотрю какие поды появились  
~~~~
kubectl get pods -n hipster-shop
NAME                                     READY   STATUS              RESTARTS   AGE
adservice-596d4d4bb7-g5gxc               0/1     ContainerCreating   0          10s
cartservice-5d4488c85d-m5nzb             0/1     ContainerCreating   0          10s
checkoutservice-7bbb5bd445-c8s2h         0/1     Running             0          10s
currencyservice-548f6ff59c-ws788         0/1     ContainerCreating   0          10s
emailservice-6d95fb4d75-krshb            0/1     ContainerCreating   0          10s
frontend-5f7bf8bcf6-vn97n                0/1     Running             0          10s
paymentservice-5b4bc8b499-rt2fw          0/1     ContainerCreating   0          10s
productcatalogservice-746f6cfbf7-98nzq   0/1     ContainerCreating   0          10s
recommendationservice-5c4c46dbd7-8s6j9   0/1     ContainerCreating   0          10s
redis-cart-57bd646894-nrkxt              0/1     ContainerCreating   0          10s
shippingservice-6dc47b7f85-6m7cf         0/1     ContainerCreating   0          10s
~~~~
Смотрю сервисы и добавляю правило на открытие порта для работы хипстершопа
https://cloud.google.com/kubernetes-engine/docs/how-to/exposing-apps#kubectl-apply_1
~~~~
gcloud compute firewall-rules create test-node-port --allow tcp:node-port
kubectl get svc -n hipster-shop
NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
adservice               ClusterIP   10.8.8.199    <none>        9555/TCP       18m
cartservice             ClusterIP   10.8.10.54    <none>        7070/TCP       18m
checkoutservice         ClusterIP   10.8.3.214    <none>        5050/TCP       18m
currencyservice         ClusterIP   10.8.2.255    <none>        7000/TCP       18m
emailservice            ClusterIP   10.8.8.52     <none>        5000/TCP       18m
frontend                NodePort    10.8.2.156    <none>        80:30898/TCP   18m
paymentservice          ClusterIP   10.8.1.240    <none>        50051/TCP      18m
productcatalogservice   ClusterIP   10.8.12.71    <none>        3550/TCP       18m
recommendationservice   ClusterIP   10.8.10.135   <none>        8080/TCP       18m
redis-cart              ClusterIP   10.8.0.30     <none>        6379/TCP       18m
shippingservice         ClusterIP   10.8.14.234   <none>        50051/TCP      18m
~~~~
~~~~
gcloud compute firewall-rules create test-node-port --allow tcp:30898
                                                                                                                                                                                                                                           Creating firewall...⠹Created [https://www.googleapis.com/compute/v1/projects/carbide-skyline-312415/global/firewalls/test-node-port].
                                                                                                                                                                                                                                            Creating firewall...done.
NAME            NETWORK  DIRECTION  PRIORITY  ALLOW      DENY  DISABLED
test-node-port  default  INGRESS    1000      tcp:30898        False
~~~~

Вынес фронт в отдельный чарт и переустановил hipster-shop  
~~~~
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop/ --namespace=hipster-shop
Release "hipster-shop" has been upgraded. Happy Helming!
NAME: hipster-shop
LAST DEPLOYED: Mon May  3 22:34:29 2021
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 2
TEST SUITE: None
~~~~


Устанавливаем ингресс и получаем сертификат ЛЕ, сайт открывается.  
~~~~
helm upgrade --install frontend kubernetes-templating/frontend --namespace hipster-shop
Release "frontend" does not exist. Installing it now.
NAME: frontend
LAST DEPLOYED: Tue May  4 13:18:03 2021
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 1
TEST SUITE: None
~~~~

~~~~
helm dep update kubernetes-templating/hipster-shop
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 1 charts
Deleting outdated charts
$ls -la kubernetes-templating/hipster-shop/charts/frontend-0.1.0.tgz 
-rw-rw-r--. 1 vivanchev vivanchev 1537 May  4 14:27 kubernetes-templating/hipster-shop/charts/frontend-0.1.0.tgz
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop/ --namespace=hipster-shop
Release "hipster-shop" has been upgraded. Happy Helming!
NAME: hipster-shop
LAST DEPLOYED: Tue May  4 14:27:46 2021
NAMESPACE: hipster-shop
STATUS: deployed
REVISION: 3
TEST SUITE: None
~~~~

Добавление редиса в качестве замисимости  
~~~~
kubernetes-templating/hipster-shop/Chart.yaml
dependencies:
  - name: frontend
    version: 0.1.0
    repository: "file://../frontend"
  - name: redis
    version: 10.5.7
    repository: "@stable"
~~~~    

Обновляем списки пакетов в репозиториях, и устанавливаем архив сабчарта в каталог charts/:  
~~~~
helm dep update kubernetes-templating/hipster-shop
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "harbor" chart repository
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "grafana" chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈Happy Helming!⎈
Saving 2 charts
Downloading redis from repo https://charts.helm.sh/stable
Deleting outdated charts
~~~~

Убеждаемся что чарт скачан и лежит в директории  
~~~~
ls -la kubernetes-templating/hipster-shop/charts/
frontend-0.1.0.tgz  redis-10.5.7.tgz    
~~~~

Для пуша ставлю плагин  
helm plugin install https://github.com/chartmuseum/helm-push.git  
Пушу 
~~~~
helm push --username admin --password Harbor12345 kubernetes-templating/hipster-shop/ otus-repochart --insecure
helm push --username admin --password Harbor12345 kubernetes-templating/frontend/ otus-repochart --insecure
~~~~

Добавил в repo.sh ссылку на репу. И ыполнил скрипт, тем самым добавил новый репозиторий.  
~~~~
helm repo list 
NAME      	URL                                                
stable    	https://charts.helm.sh/stable                      
grafana   	https://grafana.github.io/helm-charts              
jetstack  	https://charts.jetstack.io                         
harbor    	https://helm.goharbor.io                           
templating	https://harbor.34.71.66.33.nip.io/chartrepo/library
~~~~

Смотрю, что в новом подключенном репозитории (тот ято был подключен для пуша чартов - удалил)  
~~~~
helm search repo templating
NAME                   	CHART VERSION	APP VERSION	DESCRIPTION                
templating/frontend    	0.1.0        	1.16.0     	A Helm chart for Kubernetes
templating/hipster-shop	0.1.0        	1.16.0     	A Helm chart for Kubernetes
~~~~
