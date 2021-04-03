# chevvi_platform
chevvi Platform repository

1. Написан Dockerfile и на основе него собран образ.  
Образ отправлен в hub.docker.com.  
Используется образ нжинкса, в создаваемый образ копиуется конфигурация нжинкса, и homework.html, для отдачи.  
2. Написан web-prod.yaml.  
Используя этот манифест, был поднят под web.  
3. В web-prod.yaml был добавлен init container и volumes.  
Под был запущен заново (предыдущий был удален).  
Проверена работа приложения. 
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000 
4. Был собран собственный образ hipster-frontend, и запушен в hub.docker.com.  
Предпринята попытка запуска пода frontend.
Под находится в статусе Error, т.к. для работы ему не хватает переменных, которые он ожидает.
kubectl logs frontend и kubectl describe pod frontend показывают состояние пода и переменные которые ему необходимы.
5. Добавлен исправленный манифест frontend-pod-healthy.yaml, который позволяет запустить под frontend так, чтобы он был в состоянии Running.  
