# Обзор Kubernetes

1. [Кластер Kubernetes](#кластер-kubernetes)
   - [Control Plane](#control-plane)
   - [Worker Node](#worker-node)
2. [Под](#под)
3. [Преимущества Kubernetes](#преимущества-kubernetes)
4. [Недостатки Kubernetes](#недостатки-kubernetes)
5. [Управляемые сервисы Kubernetes](#управляемые-сервисы-kubernetes)
6. [ConfigMaps и Secrets](#configmaps-и-secrets)
7. [Horizontal Pod Autoscaler (HPA), автоматическое и ручное масштабирование](#масштабирование)
   - [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)
   - [Автоматическое масштабирование кластера](#автоматическое-масштабирование-кластера)
   - [Ручное масштабирование](#ручное-масштабирование)
8. [StatefulSet и Deployment](#statefulset-и-deployment)
9. [Сетевое взаимодействие в Kubernetes](#сетевое-взаимодействие-в-kubernetes)
   - [Взаимодействие подов внутри кластера](#взаимодействие-подов-внутри-кластера)
   - [IP-адреса подов и кластера](#ip-адреса-подов-и-кластера)
   - [NodePort](#nodeport)
   - [DNS для Service](#dns-для-service)
   - [LoadBalancer](#loadbalancer)
10. [Ingress и его отличия от Service](#ingress)
    - [Ingress контроллеры и ресурсы](#ingress-контроллеры-и-ресурсы)
11. [ReplicaSet и Deployment](#replicaset-и-deployment)


---
### Кластер Kubernetes <a id="кластер-kubernetes"></a>

**Кластер Kubernetes** — это набор машин, называемых узлами, которые используются для запуска контейнеризированных приложений.

#### Control Plane <a id="control-plane"></a>

Control Plane управляет состоянием кластера, создаёт поды и распределяет их между несколькими узлами. Он предоставляет RESTful API, который позволяет клиентам взаимодействовать с Control Plane.

#### Основные компоненты Control Plane:
- **etcd** — распределённое хранилище типа «ключ-значение», где хранится постоянное состояние кластера и информация о нём.
- **Scheduler** — размещает поды на рабочих узлах кластера, выбирая подходящий узел в зависимости от требуемых ресурсов пода и доступных ресурсов на узле.
- **Controller Manager** — запускает контроллеры, которые управляют состоянием кластера, например, контроллер репликации следит за тем, чтобы нужное количество реплик пода было запущено, а Deployment Controller отвечает за обновления и откаты развертываний.
- **API Server** — интерфейс для взаимодействия Controller Manager, Scheduler, etcd с узлами.

### Worker Node (Рабочий узел) <a id="worker-node"></a>

Рабочий узел содержит поды и имеет следующие компоненты:
- **Container Runtime** — отвечает за загрузку контейнеров из реестра, их запуск и остановку, а также управление их ресурсами.
- **Kubelet** — демон, который связывается с Control Plane, получает инструкции о том, какие поды запускать на узле, и поддерживает их нужное состояние.
- **Kube-Proxy** — сетевой прокси, который направляет трафик к нужным подам, а также выполняет балансировку нагрузки, распределяя трафик равномерно между всеми подами.

### Под <a id="под"></a>

Поды запускают контейнеризированные приложения и предоставляют для них общие хранилище и сетевое пространство. Поды — это минимальные единицы развертывания в Kubernetes.

---

### Преимущества Kubernetes <a id="преимущества-kubernetes"></a>

- Kubernetes обладает высокой масштабируемостью и доступностью, поддерживает функции самовосстановления, автоматического отката и горизонтального масштабирования за счёт увеличения количества узлов.

### Недостатки Kubernetes <a id="недостатки-kubernetes"></a>

1. Требует высокого уровня знаний и ресурсов для настройки и управления продакшн-средой Kubernetes.
2. Стоимость — для поддержания всех функций требуется минимальное количество ресурсов.

---

### Управляемые сервисы Kubernetes <a id="управляемые-сервисы-kubernetes"></a>

Azure, Google и Amazon предоставляют управляемые Kubernetes-сервисы: Amazon EKS, Google GKE, Azure AKS. Эти сервисы берут на себя задачи, требующие глубоких знаний, такие как настройка и конфигурирование Control Plane, масштабирование кластера и обеспечение постоянной поддержки.

---

### ConfigMaps и Secrets <a id="configmaps-и-secrets"></a>

**ConfigMaps** и **Secrets** — это объекты Kubernetes, которые используются для управления конфигурацией приложений и хранения данных, таких как пароли, ключи API и конфигурационные файлы. Они позволяют легко изменять конфигурацию без перезапуска приложения или обновления контейнера.

- Используйте ConfigMaps и Secrets в контейнерах через переменные среды или монтируйте их в качестве файлов.

```sh
kubectl apply -f app-config.yaml
```

Если поды настроены так, чтобы использовать ConfigMap или Secret, они не будут перезапущены автоматически при изменении данных. Чтобы применить изменения к подам, можно выполнить одно из следующих действий:
```sh
kubectl delete pod app-pod
kubectl rollout restart deployment app-deployment
```

Храните чувствительную информацию, такую как пароли и API-ключи, в Secrets, а не в ConfigMaps. Secrets кодируются в Base64 (это не шифрование, но защищает от случайного просмотра). Используйте RBAC (контроль доступа на основе ролей) для ограничения доступа к этим данным.

---

### Horizontal Pod Autoscaler (HPA), автоматическое и ручное масштабирование <a id="масштабирование"></a>

#### Horizontal Pod Autoscaler (HPA) <a id="horizontal-pod-autoscaler-hpa"></a>

HPA позволяет автоматически изменять количество подов в зависимости от текущей нагрузки на них. Он следит за метриками, такими как загрузка CPU и память, и при необходимости добавляет или удаляет поды для поддержания стабильной производительности.

#### Как работает HPA:
- Kubernetes использует метрики для мониторинга нагрузки. По умолчанию он отслеживает метрики CPU и памяти, но его можно настроить для работы с пользовательскими метриками.
- Когда нагрузка на поды превышает заданные пределы, HPA добавляет поды для распределения нагрузки.
- Если нагрузка снижается, HPA удаляет лишние поды для оптимизации использования ресурсов.

#### Автоматическое масштабирование кластера <a id="автоматическое-масштабирование-кластера"></a>

Cluster Autoscaler добавляет новые узлы, если поды не могут быть запущены из-за нехватки ресурсов, и удаляет узлы, если они остаются неиспользованными.

```sh
gcloud container clusters update my-cluster --enable-autoscaling --min-nodes=1 --max-nodes=5 --zone=us-central1-a
```

В этом примере Cluster Autoscaler будет поддерживать количество узлов в диапазоне от 1 до 5 в зависимости от нагрузки.

#### Ручное масштабирование <a id="ручное-масштабирование"></a>

- **Ручное масштабирование подов**: Команда `kubectl scale` позволяет изменять количество подов в Deployment, ReplicaSet или StatefulSet.
```sh
kubectl scale deployment example-deployment --replicas=5
```

- **Ручное масштабирование узлов**:
```sh
gcloud container clusters resize my-cluster --num-nodes=3 --zone=us-central1-a
```

---

### StatefulSet и Deployment <a id="statefulset-и-deployment"></a>

**StatefulSet** подходит для stateful-приложений, которые требуют уникальности, постоянного хранилища и контроля над порядком запуска, а **Deployment** — для stateless-приложений, где таких требований нет.

#### Когда использовать StatefulSet вместо Deployment?
Используйте StatefulSet, если вашему приложению требуется:
- **Уникальные идентификаторы**: Например, кластеры баз данных или кэш-серверов, которым требуются уникальные имена и идентификаторы.
- **Постоянное хранилище**: StatefulSet подходит для приложений, где данные должны сохраняться между перезапусками подов.
- **Контроль над порядком обновлений**: В ситуациях, где обновление должно выполняться строго последовательно.

#### Когда использовать Deployment?
Deployment предпочтителен для stateless-приложений, где:
- Не нужно сохранять данные на локальном хранилище.
- Все реплики идентичны, и не имеет значения, какой под обслуживает запрос.
- Масштабирование может быть выполнено параллельно без нарушения работы приложения.

---

### Сетевое взаимодействие в Kubernetes <a id="сетевое-взаимодействие-в-kubernetes"></a>

Kubernetes использует плоское сетев

ое пространство, чтобы каждый под мог взаимодействовать с другими подами внутри кластера. Это достигается через IP-адреса подов, Cluster IP, NodePort и LoadBalancer, а также Ingress для управления внешним доступом.

#### Взаимодействие подов внутри кластера <a id="взаимодействие-подов-внутри-кластера"></a>

- Под **A** может напрямую обратиться к поду **B** по его IP-адресу, если этот адрес известен.
- Если под **A** должен обращаться к поду **B** через Service (например, если под **B** может перезапуститься и поменять IP), он будет использовать DNS-имя Service.
- **Kubernetes DNS** переведёт DNS-запрос в IP-адрес, и трафик будет направлен к поду **B** через **Cluster IP**.

Если между подами включена **Network Policy**, Kubernetes проверит правила, чтобы решить, разрешён ли данный трафик.

#### IP-адреса подов и кластера <a id="ip-адреса-подов-и-кластера"></a>

- **IP-адрес пода** выделяется из диапазона адресов, заданного сетевым плагином (например, Calico, Flannel). Этот IP-адрес уникален для каждого пода и действует до тех пор, пока под существует.
- **Cluster IP** — это виртуальный IP-адрес, выделяемый каждому Service. Он обеспечивает стабильную точку доступа к набору подов и позволяет другим подам обращаться к сервису по постоянному IP.

#### NodePort <a id="nodeport"></a>

**NodePort** — это тип Service, позволяющий получить доступ к подам снаружи кластера через узлы Kubernetes:
- Открывает определённый порт на каждом узле кластера и направляет трафик на сервис, который управляет набором подов.
- Диапазон портов для NodePort по умолчанию — от 30000 до 32767. NodePort часто используют для тестирования или для простого доступа к подам без внешнего балансировщика нагрузки.

#### DNS для Service <a id="dns-для-service"></a>

Встроенный DNS-сервис Kubernetes позволяет разрешать имена Service в IP-адреса. Это упрощает взаимодействие подов, так как они могут обращаться к сервисам по именам вместо IP-адресов. Например, поды могут обращаться к сервису `my-service` по имени `my-service.default.svc.cluster.local`.

#### LoadBalancer <a id="loadbalancer"></a>

**LoadBalancer** — это тип Service, предназначенный для распределения трафика из внешней сети к подам внутри кластера:
- Работает с внешним балансировщиком нагрузки, предоставляемым облачным провайдером (например, AWS, GCP, Azure).
- Kubernetes автоматически запрашивает у облачного провайдера внешний IP и создание балансировщика для распределения трафика по узлам.
- Это делает LoadBalancer удобным для масштабируемых приложений с большим количеством пользователей.

---

### Ingress и его отличия от Service <a id="ingress"></a>

**Ingress** в Kubernetes — это ресурс, который управляет внешним доступом к сервисам внутри кластера, в основном для HTTP и HTTPS-трафика. В отличие от Service, который обеспечивает базовый доступ к подам (через Cluster IP, NodePort или LoadBalancer), **Ingress** позволяет задавать более сложные правила маршрутизации.

#### Когда использовать Ingress?
- Если необходимо гибко управлять маршрутизацией HTTP/HTTPS-трафика.
- Если требуется балансировка нагрузки по URL-путям или доменам.
- Когда нужно настроить доступ к сервисам через HTTPS с управлением SSL-сертификатами.

#### Ingress контроллеры и ресурсы <a id="ingress-контроллеры-и-ресурсы"></a>

Ingress ресурсы содержат правила маршрутизации, а Ingress контроллеры интерпретируют и исполняют эти правила:
- **Ingress ресурсы**: описывают маршруты, например, URL и домены, для направления запросов к нужным сервисам.
- **Ingress контроллеры**: обрабатывают правила и управляют сетевой маршрутизацией. Популярные Ingress контроллеры — NGINX, Traefik и контроллеры от облачных провайдеров (AWS, GCP, Azure).

### Пример Ingress ресурса

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

В этом примере:
- Запросы на `example.com/api` направляются на сервис `api-service`.
- Запросы на `example.com/web` направляются на сервис `web-service`.
### ReplicaSet и Deployment <a id="replicaset-и-deployment"></a>

`ReplicaSet` и `Deployment` действительно тесно связаны, но они выполняют разные роли в Kubernetes:

1. **ReplicaSet**:
   - `ReplicaSet` — это контроллер, который **обеспечивает постоянное количество одинаковых подов** (реплик).
   - Например, если задать `replicas: 3`, `ReplicaSet` создаст три пода и будет следить за тем, чтобы их всегда было три. Если один под упадет, `ReplicaSet` автоматически создаст новый.
   - `ReplicaSet` — это базовый механизм поддержания количества подов, но **он не управляет обновлениями подов**.

2. **Deployment**:
   - `Deployment` использует `ReplicaSet` под капотом для управления количеством подов, но добавляет **дополнительный функционал для упрощенного управления развертыванием** приложений.
   - **Основное отличие**: `Deployment` позволяет делать **rolling updates** и **rollbacks**. Это означает, что можно обновить приложение до новой версии без остановки всех подов сразу (например, обновить поды по очереди).
   - Когда создается `Deployment`, он автоматически создает `ReplicaSet` и управляет им. Если приложение нужно обновить, `Deployment` создаст новый `ReplicaSet` с новым набором подов, сохраняя старую версию до завершения обновления.

### Пример связи:

- **Представьте,** что у вас есть приложение, и вы хотите развернуть его в Kubernetes с тремя экземплярами. 
  - Вы можете создать `ReplicaSet`, указав `replicas: 3`, и Kubernetes запустит три пода.
  - Но если вам нужно обновить приложение, например, выпустить новую версию, с `ReplicaSet` это сделать сложно — придётся вручную изменять конфигурацию подов.
  - С `Deployment` это проще: вы обновляете конфигурацию `Deployment`, и он сам создаёт новый `ReplicaSet` с подами новой версии, удаляя старую версию по мере запуска новых подов.

### Итоговые различия:

- **ReplicaSet** обеспечивает **количество подов**.
- **Deployment** использует **ReplicaSet для управления числом подов**, добавляя возможность обновления и откатов, что делает его более удобным для управления версиями приложений.

### Обновленное краткое сравнение:

| Характеристика        | ReplicaSet                  | StatefulSet                    | Deployment                     |
|-----------------------|-----------------------------|---------------------------------|--------------------------------|
| Назначение            | Stateless приложения        | Stateful приложения             | Stateless приложения           |
| Порядок запуска       | Не важен                    | Гарантирован                    | Не важен                       |
| Уникальные имена      | Нет                         | Есть                            | Нет                            |
| Подключение к подам   | Случайное                   | Статическое                     | Случайное                      |
| Объем данных          | Общий для всех подов        | Отдельный для каждого пода      | Общий для всех подов           |
| Управление версиями   | Нет                         | Нет                             | Да (rollout и rollback)       |
| Контроль количества   | Да                          | Да                              | Да                             |
| Обновления и откаты   | Нет                         | Нет                             | Да                             |