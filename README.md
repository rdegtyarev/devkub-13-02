# Домашнее задание к занятию "13.2 разделы и монтирование"
Приложение запущено и работает, но время от времени появляется необходимость передавать между бекендами данные. А сам бекенд генерирует статику для фронта. Нужно оптимизировать это.
Для настройки NFS сервера можно воспользоваться следующей инструкцией (производить под пользователем на сервере, у которого есть доступ до kubectl):
* установить helm: curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
* добавить репозиторий чартов: helm repo add stable https://charts.helm.sh/stable && helm repo update
* установить nfs-server через helm: helm install nfs-server stable/nfs-server-provisioner

В конце установки будет выдан пример создания PVC для этого сервера.

### Решение

После установки nfs-server
```bash
kubectl get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
nfs-server-nfs-server-provisioner-0   1/1     Running   0          12m    10.233.94.63   node0   <none>           <none>
```

---

## Задание 1: подключить для тестового конфига общую папку
<details>

  <summary>Описание задачи</summary>  
В stage окружении часто возникает необходимость отдавать статику бекенда сразу фронтом. Проще всего сделать это через общую папку. Требования:
* в поде подключена общая папка между контейнерами (например, /static);
* после записи чего-либо в контейнере с беком файлы можно получить из контейнера с фронтом.

</details>

### Решение

1. Создаем манифест для deployment (stage) - два контейнера front и back в одном поде. Указываем и монтируем volume (stage-volume).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: stage
  name: stage
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stage
  template:
    metadata:
      labels:
        app: stage
    spec:
      containers:
        - name: front
          image: nginx
          volumeMounts:
            - mountPath: "/static"
              name: stage-volume
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
        - name: back
          image: busybox
          command: ["sleep", "3600"]
          volumeMounts:
            - mountPath: "/static"
              name: stage-volume
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
      volumes:
        - name: stage-volume
          emptyDir: {}
```
2. Применяем и проверяем:

```bash
kubectl apply -f ./task-1/
deployment.apps/stage created

kubectl get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE    IP             NODE    NOMINATED NODE   READINESS GATES
nfs-server-nfs-server-provisioner-0   1/1     Running   0          12m    10.233.94.63   node0   <none>           <none>
stage-6d6495f8f4-wj5xn                2/2     Running   0          100s   10.233.90.50   node1   <none>           <none>
```

3. Создадим файл из контейнера back и проверим результат:
```bash
kubectl exec stage-6d6495f8f4-wj5xn -c back -- sh -c "echo 'test-string' > /static/test-file.txt"

kubectl exec stage-6d6495f8f4-wj5xn -c back -- ls -la /static
total 12
drwxrwxrwx    2 root     root          4096 Apr 30 19:34 .
drwxr-xr-x    1 root     root          4096 Apr 30 19:23 ..
-rw-r--r--    1 root     root            12 Apr 30 19:34 test-file.txt

kubectl exec stage-6d6495f8f4-wj5xn -c back -- cat /static/test-file.txt
test-string
```
4. Проверяем файлы в контейнере front
```bash
kubectl exec stage-6d6495f8f4-wj5xn -c front -- ls -la /static
total 12
drwxrwxrwx 2 root root 4096 Apr 30 19:34 .
drwxr-xr-x 1 root root 4096 Apr 30 19:23 ..
-rw-r--r-- 1 root root   12 Apr 30 19:34 test-file.txt

kubectl exec stage-6d6495f8f4-wj5xn -c front -- cat /static/test-file.txt
test-string
``` 
5. Изменим файл из контейнера front
```bash
kubectl exec stage-6d6495f8f4-wj5xn -c front -- sh -c "echo 'test-string from front' > /static/test-file.txt"

kubectl exec stage-6d6495f8f4-wj5xn -c front -- cat /static/test-file.txt
test-string from front
```
5. Проверяем из контейнера back
```bash
kubectl exec stage-6d6495f8f4-wj5xn -c back -- cat /static/test-file.txt
test-string from front
```
6. Если удалить под, deployment создаст новый но volume будет пустой.
---

## Задание 2: подключить общую папку для прода
<details>

  <summary>Описание задачи</summary>  
Поработав на stage, доработки нужно отправить на прод. В продуктиве у нас контейнеры крутятся в разных подах, поэтому потребуется PV и связь через PVC. Сам PV должен быть связан с NFS сервером. Требования:
* все бекенды подключаются к одному PV в режиме ReadWriteMany;
* фронтенды тоже подключаются к этому же PV с таким же режимом;
* файлы, созданные бекендом, должны быть доступны фронту.

</details>

### Решение

Предварительно установил на worker ноды nfs-common
```bash
sudo apt install nfs-common
```

1. Подготовим манифесты для front и back (deployment). Монтируем том через persistentVolumeClaim (указываем имя pvc)
front
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: front-1
  name: front-1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: front-1
  template:
    metadata:
      labels:
        app: front-1
    spec:
      containers:
        - name: front-1
          image: nginx
          volumeMounts:
            - mountPath: "/static"
              name: prod-volume
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
      volumes:
        - name: prod-volume
          persistentVolumeClaim:
            claimName: pvc
```

back
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: back-1
  name: back-1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: back-1
  template:
    metadata:
      labels:
        app: back-1
    spec:
      containers:
        - name: back
          image: busybox
          command: ["sleep", "3600"]
          volumeMounts:
            - mountPath: "/static"
              name: prod-volume
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
            requests:
              cpu: 100m
              memory: 128Mi
      volumes:
        - name: prod-volume
          persistentVolumeClaim:
            claimName: pvc
```
2. Подготовим манифест для PersistentVolumeClaim c указанием типа ReadWriteMany. Указываем storageClassName созданный автоматически при установке NFS Provisioner service. Пример был отражен после установки.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Mi
```
3. Применяем:
```bash
kubectl apply -f task-2/
deployment.apps/back-1 created
deployment.apps/front-1 created
persistentvolumeclaim/pvc created
```
4. Проверяем
```bash
kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
back-1-6c98667d5d-lvmnb               1/1     Running   0          12m
front-1-58cf876c85-v7fj2              1/1     Running   0          12m
nfs-server-nfs-server-provisioner-0   1/1     Running   0          54m
stage-6d6495f8f4-rqkjf                2/2     Running   0          25m
```
5. Создаем файл из пода с back
```bash
kubectl exec back-1-6c98667d5d-lvmnb -- sh -c "echo 'test-string' > /static/test-file.txt"

kubectl exec back-1-6c98667d5d-lvmnb -- ls /static
test-file.txt
```

6. Проверяем файл из пода с front
```bash
kubectl exec front-1-58cf876c85-v7fj2 -- ls /static
test-file.txt

kubectl exec front-1-58cf876c85-v7fj2 -- cat /static/test-file.txt
test-string
```
7. Удалим поды front и back и проверим доступность файла после того как deployments их пересоздадут
```bash
kubectl delete pod front-1-58cf876c85-v7fj2 back-1-6c98667d5d-lvmnb 
pod "front-1-58cf876c85-v7fj2" deleted
pod "back-1-6c98667d5d-lvmnb" deleted

# получаем имена пересозданных подов
kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
back-1-6c98667d5d-bwshv               1/1     Running   0          101s
front-1-58cf876c85-dkb46              1/1     Running   0          101s
nfs-server-nfs-server-provisioner-0   1/1     Running   0          64m
stage-6d6495f8f4-rqkjf                2/2     Running   0          34m

# проверяем доступность файла
kubectl exec back-1-6c98667d5d-bwshv -- ls /static
test-file.txt

kubectl exec front-1-58cf876c85-dkb46  -- ls /static
test-file.txt
```

---
