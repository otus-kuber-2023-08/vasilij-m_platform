# vasilij-m_platform
vasilij-m Platform repository

## 1. Знакомство с Kubernetes, основные понятия и архитектура

<details>
  <summary><b>Выполнение заданий</b></summary>
  
  ### Задание 1
---
  **Вопрос:**  
    Разберитесь почему все pod в namespace `kube-system` восстановились после удаления.

  **Ответ:**  
    В нэймспейсе `kube-system` (minikube) запускаются следующие поды:  
    - coredns  
    - etcd  
    - kube-apiserver  
    - kube-controller-manager  
    - kube-proxy  
    - kube-scheduler  
    - registry  
    - registry-proxy  

  Поды `etcd`, `kube-apiserver`, `kube-controller-manager` и `kube-scheduler` являются static подами. Такие поды управляются напрямую `kubelet`'ом без участия компонентов Control Plane. `kubelet` следит за созданием/обновлением манифестов в директории `/etc/kubernetes/manifests` (по дефолту) и создает описанные в них поды без обращения к `kube-apiserver`. Собственно сам `kube-apiserver` и другие компоненты Control Plane создаются `kubelet`'ом при бутстрапе кластера (на самом деле это зависит от способа развертывания кластера, и Control Plane компоненты могут быть подняты как systemd сервисы, тогда манифесты для запуска их в виде static подов не нужны, как и сам `kubelet` на мастер-нодах).  
  При удалении этих подов из нэймспейса `kube-system` kubelet заново их поднимает, опираясь на манифесты в директории `/etc/kubernetes/manifests`.

  Под `coredns` заново поднимается после удаления, так как `coredns` является `deployment`'ом, и `deployment` контроллер следит, чтобы в кластере всегда было количество реплик, указанное в манифесте этого `deployment`.

  Поды `kube-proxy` и `registry-proxy` поднимаются, так как развернуты в кластере в составе `replicaset`, а `registry` - в составе `replicationcontroller`. Поведение этих контроллеров при удалении подов не отличается от `deployment`'а - они всегда восстанавливают в кластере то количество реплик, которое указано в их манифесте.

  ### Задание 2
---
  **Результат выполнения**
  
  1. Написан Dockerfile, запускающий web-сервер NGINX на порту `8000`, отдающий содержимое директории `/app` внутри контейнера (например, если в директории `/app` лежит файл `homework.html`, то при запуске контейнера данный файл должен быть доступен по URL `http://localhost:8000/homework.html`) и работающий с UID `1001`.   
  Dockerfile и конфиг для NGINX находится в директории `kubernetes-intro/web`. Образ собран и загружен в Docker Hub под тегом `vasiilij/nginx:k8s-intro`.

  2. Создан манифест `kubernetes-intro/web-pod.yaml` для запуска пода с контейнером на основе образа `vasiilij/nginx:k8s-intro`.

  3. В под к основному контейнеру добавлен init контейнер, генерирующий страницу `index.html`.

  4. Работоспособность приложения проверена (скриншот ниже):
  ![index.html content](./screens/1.2.1.jpg)

  ### Hipster Shop | Задание со *
---
  **Результат выполнения**
  
  1. Причиной, по которой падал pod `frontend`, было отсутствие переменных окружения, необходимых для работы приложения.

  2. Создан манифест `kubernetes-intro/frontend-pod-healthy.yaml`, в котором для контейнера `frontend` указаны необходимые переменные окружения.

</details>

## 2. Kubernetes controllers. ReplicaSet, Deployment, DaemonSet

<details>
  <summary><b>Выполнение заданий</b></summary>
  
  ### Задание 1 (Обновление ReplicaSet)
---
  **Вопрос:**  
    Почему обновление ReplicaSet не повлекло обновление запущенных pod?

  **Ответ:**  
    После изменения в манифесте ReplicaSet версии образа для контейнера `frontend` и применения этого манифеста, в кластере остались запущены поды со старой версией приложения, то есть новые поды не запустились вместо уже запущенных. Это произошло потому, что ReplicaSet следит только за тем, чтобы количество подов в кластере с определёнными лейблами (эти лейблы указаны в селекторе ReplicaSet), совпадало с числом реплик в поле `.spec.replicas`.  
    На момент изменения версии образа в спецификации ReplicaSet, количество запущенных контейнеров с лейблом `app: frontend` уже равнялось трём, поэтому ReplicaSet не стало пересоздавать новые поды с обновленной версией приложения.

  ### Задание 2 (Deployment | Задание со *)
---
  **Результат выполнения**
  
  ***Реализация аналога blue-green развертывания:***
  1. Развертывание трех новых pod
  2. Удаление трех старых pod

  Blue-green развертывание можно реализовать следующими параметрами секции `.spec` в манифесте deployment:

  ```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 100%        # сразу будут подняты все реплики с новой версией приложения (maxSurge: 100%),
      maxUnavailable: 0     # при этом старые реплики будут удалены только после того, как новые реплики будут готовы (maxUnavailable: 0)
  ```

  Весь манифест находится в файле `kubernetes-controllers/paymentservice-deployment-bg.yaml`.

  ***Реализация аналога Reverse Rolling Update развертывания:***
  1. Удаление одного старого pod
  2. Создание одного нового pod
  3. …

  Reverse Rolling Update развертывание можно реализовать следующими параметрами секции `.spec` в манифесте deployment:

  ```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0          # новая реплика приложения поднимется только после того (maxSurge: 0),
      maxUnavailable: 1    # как одна старая будет удалена (maxUnavailable: 1), и так далее, пока все старые реплики не будут заменены новыми
  ```

  Весь манифест находится в файле `kubernetes-controllers/paymentservice-deployment-reverse.yaml`.

  ### Задание 3 (Probes)
---
  **Результат выполнения**
  
  Создан манифест `kubernetes-controllers/frontend-deployment.yaml`, в котором добавлена Readiness Probe для периодеческого опроса эндпойнта `/_healthz` для контейнера `server`.

  ### Задание 4 (DaemonSet | Задание со *)
---
  **Результат выполнения**
  
  В кластере был развернут Node Exporter в виде DaemonSet.

  За основу манифеста `kubernetes-controllers/node-exporter-daemonset.yaml` был взят сгенерированный манифест `daemonset.yaml` из шаблона helm-чарта `prometheus-community/prometheus-node-exporter` командой `helm template`.

  На скриншоте показано, что поды с Node Exporter запустились только на worker нодах:
  ![index.html content](./screens/2.4.1.jpg)

  Для проверки, что метрики отдаются, после применения манифеста `node-exporter-daemonset.yaml` необходимо:
  1. Пробросить порт в любой под с Node Exporter: `kubectl port-forward prometheus-node-exporter-6gqrn 9100:9100`
  2. Запросить метрики командой `curl localhost:9100/metrics` или открыть в браузере адрес http://localhost:9100/metrics

  ### Задание 5 (DaemonSet | Задание со **)
---
  **Результат выполнения**

  Для запуска подов с Node Exporter на Control plane нодах в манифест `kubernetes-controllers/node-exporter-daemonset.yaml` необходимо добавить параметры `tolerations` в `.spec.template.spec`:

  ```yaml
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
  ```

  Это укажет планировщику Kubernetes Scheduler для подов с Node Exporter игнорировать taint `node-role.kubernetes.io/control-plane:NoSchedule`, который добавлен на Control plane ноды кластера.

  После применения манифеста `kubernetes-controllers/node-exporter-daemonset.yaml` с параметрами tolerations под с Node Exporter окажется запущен также и на Control plane ноде:
  ![index.html content](./screens/2.5.1.jpg)

</details>

## 3. Настройка сетевой связности для приложения. Добавление service, ingress. Установка MetalLB

<details>
  <summary><b>Выполнение заданий</b></summary>

  ### Задание 1 (Добавление проверок Pod)
---
  **Вопрос:**  
  Почему следующая конфигурация валидна, но не имеет смысла?
  
  ```yaml
  livenessProbe:
    exec:
      command:
        - 'sh'
        - '-c'
        - 'ps aux | grep my_web_server_process'
  ```

  **Ответ:**  
  Код возврата данной команды всегда будет равен 0, вследствие чего данная livenessProbe всегда будет успешно проходить. Возможно такая проверка будет иметь смысл, если `my_web_server_process` не является основным процессом в поде (то есть его PID не равен 1), но он должен быть запущен в поде после основного. Тогда в этом случае необходимо добавить в команду дополнительную обработку, чтобы `grep` возвращал код `1`, если процесса `my_web_server_process` нет среди запущенных.


  ### Установка MetalLB
---
  MetalLB позволяет запустить внутри кластера L4-балансировщик, который будет принимать извне запросы к сервисам и раскидывать их между подами.  
  
  Для его установки нужно:
  
  1. Включить `IPVS` в `kube-proxy`, отредактировав `kube-proxy` configmap:
  ```bash
  kubectl edit configmap -n kube-system kube-proxy
  ```

  ```yaml
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  mode: "ipvs"
  ipvs:
    strictARP: true
  ```

  2. Установить Metallb, применив манифест: 
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
  ```

  3. Далее нужно определить пул IP адресов, которые MetalLB будет назначать сервисам с типом `LoadBalancer`. Сделать это можно, создав ресурс с типом `IPAddressPool` (для Layer 2 режима также нужно создать ресурс `L2Advertisement`):
  ```bash
  kubectl apply -f metallb-config.yaml
  ```

  ```yaml
  ---
  apiVersion: metallb.io/v1beta1
  kind: IPAddressPool
  metadata:
    name: default
    namespace: metallb-system
  spec:
    addresses:
    - 172.17.255.1-172.17.255.255

  ---
  apiVersion: metallb.io/v1beta1
  kind: L2Advertisement
  metadata:
    name: minikube
    namespace: metallb-system
  spec:
    ipAddressPools:
    - default
  ```

  ### MetalLB | Проверка конфигурации
---
  1. Применим манифест `./kubernetes-networks/web-svc-lb.yaml`, который создаст сервис с типом `Loadbalancer`, после чего увидим, что MetalLB назначил нашему сервису IP адрес (`EXTERNAL-IP`) из пула `default`:
  ```bash
  $ kubectl get svc web-svc-lb 
  NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
  web-svc-lb   LoadBalancer   10.105.40.111   172.17.255.1   80:32694/TCP   6s  
  ```

  2. Добавим маршрут в нашей зостовой ОС до подсети `172.17.255.0/24` через IP адрес `Minikube`:
  ```bash
  sudo ip route add 172.17.255.0/24 via 192.168.49.2
  ```
  
  3. Далее можно пройти в браузере на страницу `http://172.17.255.1/index.html` и убедиться, что наше приложение работает.


  ### Задание со * | DNS через MetalLB
---
  В манифесте `./kubernetes-networks/coredns/dns-svc-lb.yaml` описаны два сервиса с типом `Loadbalancer`. Эти сервисы после создания позволяют обращаться к внутрикластернему DNS (CoreDNS) из внешней сети. Так как Kubernetes в настоящее время не поддерживает мультипротокольные сервисы LoadBalancer, то для каждого протокола (TCP и UDP) необходимо создать свой сервис. Но чтобы этим сервисам был назначен один и тот же IP адрес, нужно в аннотации `metallb.universe.tf/allow-shared-ip` указать одинаковый общий ключ.

  После применения манифеста `./kubernetes-networks/coredns/dns-svc-lb.yaml` обоим сервисам будет назначен одинаковый IP адрес:
  ```bash
  $ kubectl get svc -n kube-system                 
  NAME             TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                  AGE
  dns-svc-lb-tcp   LoadBalancer   10.99.195.178    172.17.255.2   53:31644/TCP             13s
  dns-svc-lb-udp   LoadBalancer   10.102.217.198   172.17.255.2   53:30531/UDP             13s
  ```

  Теперь мы можем получить IP адрес, назначенный какому-либо сервису, обратившись к CoreDNS кластера, например:
  ```bash
  $ nslookup web-svc-lb.default.svc.cluster.local 172.17.255.2
  Server:		172.17.255.2
  Address:	172.17.255.2#53

  Name:	web-svc-lb.default.svc.cluster.local
  Address: 10.105.40.111  
  ```

  ### Создание Ingress
---
  1. Для установки NGINX ingress контроллера применим манифест:
  ```bash
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/baremetal/deploy.yaml
  ```

  Далее дождемся запуска пода с контроллером:
  ```
  $ kubectl get pods -n ingress-nginx -w 
  NAME                                        READY   STATUS      RESTARTS   AGE
  ingress-nginx-admission-create-gzcwm        0/1     Completed   0          2m3s
  ingress-nginx-admission-patch-7xv82         0/1     Completed   0          2m3s
  ingress-nginx-controller-79bc9f5df8-l82wx   1/1     Running     0          2m3s
  ```
  
  2. Создадим `LoadBalancer` сервис `ingress-nginx`:
  ```bash
  kubectl apply -f nginx-lb.yaml
  ```

  Проверим, что MetalLB назначил сервису IP адрес:
  ```bash
  $ kubectl -n ingress-nginx get svc                                                                        
  NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
  ingress-nginx                        LoadBalancer   10.110.20.187   172.17.255.3   80:31497/TCP,443:30753/TCP   17s
  ingress-nginx-controller             NodePort       10.98.0.50      <none>         80:30747/TCP,443:31679/TCP   7m38s
  ingress-nginx-controller-admission   ClusterIP      10.109.153.7    <none>         443/TCP                      7m38s
  ```
  
  3. Создадим Headless-сервис для проксирования запросов в наше приложение. Headless-сервис - это просто А-запись в CoreDNS, т.е. имя сервиса преобразуется не в виртуальный IP (как раз его нет - `clusterIP: None` в манифесте), а сразу в IP нужного пода. Применим манифест `./kubernetes-networks/web-svc-headless.yaml` и убедимся, что ClusterIP действительно не был назначен:
  ```bash
  kubectl apply -f web-svc-headless.yaml
  ```

  ```bash
  $ kubectl get svc web-svc               
  NAME      TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
  web-svc   ClusterIP   None         <none>        80/TCP    54s  
  ```

  4. Создадим ресурс `Ingress` из манифеста `./kubernetes-networks/web-ingress.yaml` для того, чтобы в конфигурации ingress-контроллера появились нужные правила:
  ```bash
  kubectl apply -f web-ingress.yaml
  ```

  Проверим, что корректно заполнены Address и Backends:
  ```bash
  $ kubectl describe ingress/web
  Name:             web
  Labels:           <none>
  Namespace:        default
  Address:          192.168.49.2
  Ingress Class:    <none>
  Default backend:  <default>
  Rules:
    Host        Path  Backends
    ----        ----  --------
    *           
                /web(/|$)(.*)   web-svc:8000 (10.244.0.4:8000,10.244.0.5:8000,10.244.0.6:8000)
  Annotations:  kubernetes.io/ingress.class: nginx
                nginx.ingress.kubernetes.io/rewrite-target: /$2
  Events:
    Type    Reason  Age                  From                      Message
    ----    ------  ----                 ----                      -------
    Normal  Sync    54s (x4 over 5m52s)  nginx-ingress-controller  Scheduled for sync
  ```

  5. Теперь можно проверить, что приложение доступно в браузере по адресу `http://172.17.255.3/web/index.html`.


  ### Задание со * | Ingress для Dashboard
---
  1. Для установки Dashboard применим манифесты из директории `./kubernetes-networks/dashboard` (за основу манифестов взят [официальный манифест](https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml) по установке Dashboard):
  ```bash
  kubectl apply -f dashboard/namespace.yaml && sleep 2 && kubectl apply -f dashboard
  ```

  2. Для доступа к Dashboard через Ingress-контроллер (через префикс `/dashboard`) описан Ingress ресурс в манифесте `./kubernetes-networks/dashboard/ingress.yaml`

  3. После создания всех русурсов в кластере необходимо получить токен от сервис-аккаунта `admin-user`. Сделать это можно командой:
  ```bash
  kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
  ```
  
  4. Далее можно выполнить вход в Dasboard в браузере на странице `https://172.17.255.3/dashboard`, введя полученный токен:
  ![index.html content](./screens/3.1.jpg)


  ### Задание со * | Canary для Ingress
---
  Для реализации канареечного развертывания с помощью ingress-nginx были написаны манифесты в директории `./kubernetes-networks/canary`. После их применения в нэймспейсе `canary` будут созданы ресурсы для `app-main` и `app-canary` приложений, которые по факту являются веб-серверами NGINX, отдающими страницу, содержащую имя хоста, IP-адрес и порт, а также URI запроса и местное время веб-сервера.

  1. Применим манифесты:
  ```bash
  kubectl apply -f canary/namespace.yaml && sleep 2 && kubectl apply -f canary
  ```

  2. Проверим созданные ресурсы:
  ```bash
  $ kubectl -n canary get all && kubectl -n canary get ingress
  NAME                             READY   STATUS    RESTARTS   AGE
  pod/app-canary-86fdf78c8-jjxfw   1/1     Running   0          5m3s
  pod/app-main-5857f664f-c2g7q     1/1     Running   0          5m3s

  NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
  service/app-canary   ClusterIP   None         <none>        8000/TCP   5m3s
  service/app-main     ClusterIP   None         <none>        8000/TCP   5m3s

  NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/app-canary   1/1     1            1           5m3s
  deployment.apps/app-main     1/1     1            1           5m3s

  NAME                                   DESIRED   CURRENT   READY   AGE
  replicaset.apps/app-canary-86fdf78c8   1         1         1       5m3s
  replicaset.apps/app-main-5857f664f     1         1         1       5m3s
  NAME     CLASS   HOSTS   ADDRESS        PORTS   AGE
  canary   nginx   *       192.168.49.2   80      19s
  main     nginx   *       192.168.49.2   80      19s
  ```

  3. Ingress с canary развертыванием описан в файле `./kubernetes-networks/canary/ingress-canary.yaml`. Благодаря этому Ingress трафик с заголовком `canary` будет попадать на приложение `app-canary`, в то время как трафик без этого заголовка пойдет на приложение `app-main`:

  Сначала отправим запросы с хедером `canary` с различными значениями и убедимся, что ответ приходит от приложения `app-canary`:
  ```bash
  $ curl -s -H "canary: 1" http://172.17.255.3/canary 
  Server address: 10.244.0.21:8080
  Server name: app-canary-6684864d55-4b2nn
  Date: 25/Sep/2023:21:16:18 +0000
  URI: /
  Request ID: 2829ee866139dfbca468fc729d4c6296

  $ curl -s -H "canary: 2" http://172.17.255.3/canary
  Server address: 10.244.0.21:8080
  Server name: app-canary-6684864d55-4b2nn
  Date: 25/Sep/2023:21:16:22 +0000
  URI: /
  Request ID: 81b44c0e64e23c6d81c644523e5a2457

  $ curl -s -H "canary: test" http://172.17.255.3/canary
  Server address: 10.244.0.21:8080
  Server name: app-canary-6684864d55-4b2nn
  Date: 25/Sep/2023:21:16:27 +0000
  URI: /
  Request ID: bb54bb77012a593d1d00e6cc251fb2fa
  ```

  Теперь отправим запросы с любым хедером и без него и убедимся, что ответ приходит от приложения `app-main`:
  ```bash
  $ curl -s -H "sparrow: captain" http://172.17.255.3/canary
  Server address: 10.244.0.22:8080
  Server name: app-main-8bbb965c4-fbpzp
  Date: 25/Sep/2023:21:19:01 +0000
  URI: /
  Request ID: 1c384632a303ac7b2bca1cb218f1f3f4

  $ curl -s -H "x: y" http://172.17.255.3/canary
  Server address: 10.244.0.22:8080
  Server name: app-main-8bbb965c4-fbpzp
  Date: 25/Sep/2023:21:19:11 +0000
  URI: /
  Request ID: 5be2fa5ba4f27a7b44c1bdae66c14814

  $ curl -s http://172.17.255.3/canary 
  Server address: 10.244.0.22:8080
  Server name: app-main-8bbb965c4-fbpzp
  Date: 25/Sep/2023:21:19:17 +0000
  URI: /
  Request ID: a73ea4beff282ae51e753fee31821d4f
  ```
</details>

## 4. Настройка сервисных аккаунтов и ограничение прав для них

<details>
  <summary><b>Выполнение заданий</b></summary>

  ### Задание 1
---
  1. Создать Service Account `bob` , дать ему роль `admin` в рамках всего кластера
  2. Создать Service Account `dave` без доступа к кластеру

  **Результат выполнения**  

  1. Service Account `bob` и ClusterRoleBinding `admin-clusterrole` описаны в манифестах `./kubernetes-security/task01/01-sa-bob.yaml` и `./kubernetes-security/task01/02-clusterrolebinding-bob.yaml`.
  2. Service Account `dave` описан в манифесте `./kubernetes-security/task01/03-sa-dave.yaml`. Чтобы у `dave` не было доступа к кластеру достаточно просто не привязывать его к какой-либо роли через объекты RoleBinding/ClusterRoleBinding.

  ### Задание 2
---
  1. Создать Namespace `prometheus`
  2. Создать Service Account `carol` в этом Namespace
  3. Дать всем Service Account в Namespace `prometheus` возможность делать `get`, `list`, `watch` в отношении Pods всего кластера

  **Результат выполнения**

  1. Namespace `prometheus` описано в манифесте `./kubernetes-security/task02/01-namespace.yaml`.
  2. Service Account `carol` описан в манифесте `./kubernetes-security/task02/02-sa-carol.yaml`.
  3. Чтобы все сервисные аккаунты в Namespace `prometheus` имели возможность делать `get`, `list`, `watch` в отношении Pods всего кластера, нужно применить следующие манифесты:
     1.  `./kubernetes-security/task02/03-clusterrole-pods-viewer.yaml` - описывает ClusterRole `pods-viewer`
     2.  `./kubernetes-security/task02/04-clusterrolebinding-pods-viewer.yaml` - описывает ClusterRoleBinding `serviceaccounts-pods-viewer` (привязывает сервисные аккаунты из Namespace `prometheus` к ClusterRole `pods-viewer`).

  ### Задание 3
---
  1. Создать Namespace `dev`
  2. Создать Service Account `jane` в Namespace `dev`
  3. Дать `jane` роль `admin` в рамках Namespace `dev`
  4. Создать Service Account `ken` в Namespace `dev`
  4. Дать `ken` роль `view` в рамках Namespace `dev`

  **Результат выполнения**
  
  1. Namespace `dev` описано в манифесте `./kubernetes-security/task03/01-namespace.yaml`.
  2. Service Account `jane` описан в манифесте `./kubernetes-security/task03/02-sa-jane.yaml`.
  3. Манифест `./kubernetes-security/task03/03-rolebinding-jane.yaml` - описывает RoleBinding `jane-admin` в рамках Namespace `dev` (привязывает сервисный аккаунт `jane` из Namespace `dev` к ClusterRole `admin`).
  4. Service Account `ken` описан в манифесте `./kubernetes-security/task03/04-sa-ken.yaml`.
  5. Манифест `./kubernetes-security/task03/05-rolebinding-ken.yaml` - описывает RoleBinding `ken-view` в рамках Namespace `dev` (привязывает сервисный аккаунт `ken` из Namespace `dev` к ClusterRole `view`).

</details>
