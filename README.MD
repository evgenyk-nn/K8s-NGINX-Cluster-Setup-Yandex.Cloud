#Создание кластера Kubernetes, группы узлов. Развертывание в кластере приложения — веб-сервер NGINX. Балансировка нагрузки.

##Создание кластера Kubernetes и группы узлов в нём.
##Развертывание в кластере приложения — веб-сервер NGINX.

Основное средство взаимодействия с кластером — инструмент [kubectl](https://kubernetes.io/ru/docs/reference/kubectl/kubectl/). Установка по [инструкции](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/).

В консоли управления входим в созданный кластер Managed Service for Kubernetes и нажимаем кнопку **Подключиться**. В открывшемся окне скопируем команду для подключения:

    ```
     yc managed-kubernetes cluster get-credentials \
       --id <идентификатор_кластера> \
       --external
    ```

    Для проверки правильности установки и подключения, просматриваем конфигурацию:

    ```
     kubectl config view
    ```

    Ответ должен получится примерно таким :

    ```
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: DATA+OMITTED
        server: https://178.154.206.242
    name: yc-managed-k8s-cat2oek6hbp7mnhhhr4m
    contexts:
    ...
    ```

### Выполнение манифеста

1. Для создания или обновления ресурсов в кластере используется команда `apply`. Файл манифеста указывается после флага `f`.

   ```
    kubectl apply -f <путь_к_файлу_my-nginx.yaml>
   ```

   Если результат будет успешным, вы увидите сообщение:

   ```
    deployment.apps/my-nginx-deployment created
   ```

2. Чтобы убедиться, что приложение создано, посмотрите список подов:

   ```
    kubectl get pods
   ```

   Дожидаемся статуса `Running`:

   ```
    NAME                                   READY   STATUS    RESTARTS   AGE
    my-nginx-deployment-65b9b678b6-zmfww   1/1     Running   0          5m27s
   ```

   Теперь получаем более подробную информацию, выполнив ту же команду с флагом `-o wide`:

   ```
    kubectl get pods -o wide
   ```

   Видим внутренний IP-адрес, который присвоен поду. Это пригодится, если нужно узнать, где именно развёрнуто приложение.

   Чтобы получить максимально подробную информацию о запущенном приложении, используем команду `describe`:

   ```
    kubectl describe deployment/my-nginx-deployment
   ```

### Масштабирование

1. Теперь увеличьте количество подов. Вручную это можно сделать двумя способами:
   - изменить файл манифеста, указав в директиве `replicas` нужное число подов, и снова выполнить команду `apply`;
   - если файла манифеста нет под рукой — использовать команду `scale`:
     ```
     kubectl scale --replicas=3 deployment/my-nginx-deployment
     ```

Если всё получится, в выводе команды `kubectl get pods` вы увидите сообщение:

```
NAME                                   READY   STATUS    RESTARTS   AGE
my-nginx-deployment-65b9b678b6-6whpp   1/1     Running   0          117s
my-nginx-deployment-65b9b678b6-wtph9   1/1     Running   0          117s
my-nginx-deployment-65b9b678b6-zmfww   1/1     Running   0          14m
```

### Кластер как код

Управление кластерами Kubernetes отлично вписывается в концепцию Infrastructure as Code: вы можете описать конфигурацию кластера в текстовом файле — манифесте. Также можно разворачивать кластеры Kubernetes с помощью [Terraform](https://registry.tfpla.net/providers/yandex-cloud/yandex/latest/docs/resources/kubernetes_cluster).

## Балансировка нагрузки

1. Создание файла-манифест `load-balancer.yaml`:

   ```
    apiVersion: v1
    kind: Service
    metadata:
      name: my-loadbalancer
    spec:
      selector:
         app: nginx
      ports:
      - port: 80
        targetPort: 80
      type: LoadBalancer
   ```

   Где:

   `port` — порт сетевого балансировщика, на котором будут обслуживаться пользовательские запросы;

   `targetPort` — порт контейнера, на котором доступно приложение;

   `selector` — метка селектора из шаблона подов в манифесте объекта `Deployment`.

2. Выполнение манифеста:

   ```
    kubectl apply -f <путь_к_файлу_load-balancer.yaml>

   ```

   Вы увидите сообщение:

   ```
    service/my-loadbalancer created
   ```

3. В консоли управления откройте раздел **Load Balancer**. Там должен появиться балансировщик нагрузки с префиксом k8s в имени и уникальным идентификатором кластера Kubernetes.
4. Скопируйте IP-адрес балансировщика в адресную строку браузера. Вы увидите приветственную страницу NGINX.
