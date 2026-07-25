### Кластер создан с помощью kubeadm
![[Pasted image 20260725140632.png]]
### Узлы имеют уникальные hostname, MAC address
**kube1**:
![[Pasted image 20260725140819.png]]**kube2**:
![[Pasted image 20260725141059.png]]
**kube3**:
![[Pasted image 20260725141001.png]]
### Готовность узлов после установки CNI
![[Pasted image 20260725141458.png]]
### Работоспособность CoreDNS
Создадим `pod` с именем `curl-pod` (имеется набор утилит для отладки, в том числе и `nslookup`).
**Созданный curl-pod в пространстве имен homework**:
![[Pasted image 20260725143018.png]]
На момент выполнения данной работы уже были созданы сервисы для `frontend`-а и `backend`-а, поэтому для проверки CoreDNS воспользуемся соотв. доменными именами (не FQDN, так как `curl-pod` находится в том же пространстве имен, что и одноименные сервисы).
**Проверка CoreDNS**:
![[Pasted image 20260725143335.png]]
### Проверка достижимости Pod-ов
Для проверки достижимости произведем, например, `http`-запрос от `frontend` к `backend`(в качестве доменного имени будем использовать доменное имя `backend`-сервиса).
**Результат обращения frontend-а к backend-у**:
![[Pasted image 20260725143935.png]]
### Версии kubeadm, kubelet, containerd
![[Pasted image 20260725144909.png]]
### Backend работает в трех репликах
![[Pasted image 20260725145129.png]]
### Вывод имени Pod-а и версии при обращении к backend-у
![[Pasted image 20260725145537.png]]
### Frontend работает в двух репликах
![[Pasted image 20260725145637.png]]
### Доказательство запуска PostgreSQL через StatefulSet
Сделаем запрос к `/api/visits` и отобразим номер `ID`. Затем удалим `pod` postgres-0 (StatefulSet обязан его восстановить) и проверим `ID` при новом обращении к `/api/visits`(номер должен будет уменьшиться на 1).
**Первое обращение к \/api\/visits**:
![[Pasted image 20260725154546.png]]
**Удаление postgres-0**:
![[Pasted image 20260725154647.png]]
**Обращение к \/api\/visits**:
![[Pasted image 20260725154814.png]]
### Объекты EndpointSlice созданные с объектами Service
![[Pasted image 20260725155015.png]]

### Объекты типа ingress, созданные в рамках кластера
**Объект типа Ingress**:
![[Pasted image 20260725155257.png]]
**Объект типа Service(NodePort) созданный для внешнего доступа**:
![[Pasted image 20260725155454.png]]
**Обращение к kube-lab.local**:
![[Pasted image 20260725155635.png]]
### Связь объектов БД с Configmap и Secret
**Содержимое homework-database-stateful-set.yaml**
![[Pasted image 20260725155821.png]]
### Удаление backend pod
**Удаление и вывод информации о вновь созданном pod-е**:
![[Pasted image 20260725163247.png]]
Видно, что новый Pod был создан объектом `ReplicaSet`. Вообще создавая `Deployment`, происходит автоматическое создание объекта `ReplicaSet`, и в сою очередь объекты типа `ReplicaSet` отвечают за поддержание кол-ва Pod-ов. По этой причине в `Controlled By` содержится объект `ReplicaSet`.
### Rollout и Rollback
**Замена на новый образ:**
![[Pasted image 20260725171942.png]]
**Версия образа**:
![[Pasted image 20260725172448.png]]
Откатим все изменения назад.
После команды `rollout back` получаем два объекта `ReplicaSet`, один из которых относится к `homework-backend:v2`(0 pod-ов), а другой к `truncheon/homework-backend:v1`(3 pod-а).
**Вывод объектов ReplicaSet**:
![[Pasted image 20260725173001.png]]
### Работоспособность NetworkPolicy
Легче всего проверить работу NetworkPolicy можно на примере протокола DNS. На данном этапе имеется политика, разрешающая DNS-запросы.
**Вывод DNS запроса при разрешающей политике**:
![[Pasted image 20260725174753.png]]
Удалим из `etcd` соотв. `NetworkPolicy`.
**Вывод DNS запроса при отсутствии политики**:
![[Pasted image 20260725175006.png]]

### Readiness
Сделаем специально ошибки в манифесте `backend` deployment-а.
![[Pasted image 20260725175753.png]]
Видно, что состояние у трех подов Ready, но на скриншоте кол-во "перезагрузок" равно 3.
![[Pasted image 20260725175942.png]]
Всё это происходит, потому что в качестве пути указан неверный endpoint.
**Различие**: "Running" от "Ready" отличается тем, что в состоянии "Running" основной процесс pod-а работает, но сам он не прошел проверку `readinessProbe`, которая явно прописана в манифесте `Pod`-а, `ReplicaSet`-а или `Deployment`-а.