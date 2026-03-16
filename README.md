# Домашнее задание к занятию «Обновление приложений»

## ` Дмитрий Климов `

## Цель задания
### `Выбрать и настроить стратегию обновления приложения.`

## Чеклист готовности к домашнему заданию
### `Кластер K8s.`

## Задание 1. Выбрать стратегию обновления приложения и описать ваш выбор

1. Имеется приложение, состоящее из нескольких реплик, которое требуется обновить.
2. Ресурсы, выделенные для приложения, ограничены, и нет возможности их увеличить.
3. Запас по ресурсам в менее загруженный момент времени составляет 20%.]
4. Обновление мажорное, новые версии приложения не умеют работать со старыми.
5. Вам нужно объяснить свой выбор стратегии обновления приложения.


## Ответ:

### Исходные данные:
- **Количество реплик:** Несколько.
- **Ограничение ресурсов:** Ресурсы строго ограничены, увеличить их нельзя.
- **Запас ресурсов:** 20% в периоды минимальной нагрузки.
- **Тип обновления:** Мажорное.
- **Совместимость:** Новая версия **несовместима** со старой (не могут работать одновременно).

---

### Выбранная стратегия: `Recreate` (Пересоздание)

### Обоснование выбора:

Для данной ситуации единственным верным решением является стратегия **Recreate**. Ниже приведен подробный анализ, почему другие популярные стратегии не подходят, и почему `Recreate` является оптимальной:

1.  **Проблема несовместимости версий (Критический фактор):**
    По условию, новые версии не умеют работать со старыми. Это означает, что при использовании стратегий `RollingUpdate` или `Canary`, когда в кластере одновременно находятся поды обеих версий, приложение будет работать некорректно (ошибки в БД, конфликты API, рассинхронизация сессий). Стратегия **Recreate** гарантирует, что все старые поды будут удалены до того, как начнут подниматься новые, исключая их одновременную работу.

2.  **Жесткое ограничение ресурсов:**
    - Стратегия **Blue/Green** требует удвоения ресурсов (нужно держать две полные копии приложения), что невозможно по условию.
    - Стратегия **RollingUpdate** с параметром `maxSurge > 0` также требует дополнительных ресурсов для запуска новых подов перед удалением старых.
    - У нас есть запас всего 20% в моменты низкой нагрузки, чего недостаточно для полноценного параллельного развертывания новой версии без деградации сервиса.

3.  **Использование запаса ресурсов:**
    Наличие 20% запаса ресурсов в менее загруженное время позволяет быстрее поднять новые поды после удаления старых, так как узлы кластера не будут перегружены в момент массового старта новых контейнеров.

4.  **Компромисс (Downtime):**
    Главный минус стратегии `Recreate` — это наличие периода недоступности (downtime) во время обновления. Однако, учитывая полную несовместимость версий и отсутствие ресурсов для Blue/Green, простой является необходимым и оправданным риском для сохранения целостности данных и корректной работы приложения.

---

### Пример манифеста Deployment со стратегией Recreate:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 5
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: my-registry/my-app:v2.0.0
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "400m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

## Резюме:

```
В условиях несовместимости версий и отсутствия свободных ресурсов для запуска параллельных копий приложения, стратегия Recreate является единственным безопасным способом провести обновление, минимизируя риски возникновения логических ошибок в работе приложения за счет кратковременного планового простоя.
```

## Задание 2. Обновить приложение
   1. Создать deployment приложения с контейнерами nginx и multitool. Версию nginx взять 1.19. Количество реплик — 5.
   2. Обновить версию nginx в приложении до версии 1.20, сократив время обновления до минимума. Приложение должно быть            доступно.
   3. Попытаться обновить nginx до версии 1.28, приложение должно оставаться доступным.
   4. Откатиться после неудачного обновления.

## Ответ:

### 1. Создание исходного Deployment
Был создан манифест с 5 репликами, включающий контейнеры `nginx:1.19` и `network-multitool`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-multitool
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx-multitool
  template:
    metadata:
      labels:
        app: nginx-multitool
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
      - name: multitool
        image: wbitt/network-multitool
        env:
        - name: HTTP_PORT
          value: "8080"
        ports:
        - containerPort: 8080
```

![kubectl apply -f deployment.yaml](https://github.com/Dmitriy-py/Updating-applications/blob/7b1531a3b999b6e3916763a70871bd924a70ad90/kubectl%20apply%20.png)

### 2. Обновление до Nginx 1.20

```bash
kubectl set image deployment/nginx-multitool nginx=nginx:1.20
```
Обновление прошло успешно благодаря стратегии RollingUpdate (по умолчанию), приложение оставалось доступным.

### 3. Обновление до Nginx 1.28 (неудачное)

Попытка обновить на несуществующую версию:
```bash
kubectl set image deployment/nginx-multitool nginx=nginx:1.28
```
Результат: Часть подов перешла в статус ImagePullBackOff. Однако, так как новые поды не прошли проверку готовности, Kubernetes не удалил все старые работающие поды (версии 1.20), и приложение продолжало отвечать на запросы.

![kubectl set image deployment/nginx-multitool nginx=nginx](https://github.com/Dmitriy-py/Updating-applications/blob/90f50d45aa3cd5466b73c4243ea8a840927ad65a/kubectl_set_image.png)


## 4. Откат обновления

Для возврата к стабильной версии был выполнен откат:

```bash
kubectl rollout undo deployment/nginx-multitool
```
Проверка итоговой версии:

```dash
kubectl describe pod <pod-name> | grep Image:
```
### Результат подтвердил возврат к образу nginx:1.20.

![kubectl describe pod <pod-name> | grep Image](https://github.com/Dmitriy-py/Updating-applications/blob/551b8b9cb7d133ed7df503b34d1f503fe98a6882/kubectl_describe_pod.png)


## Задание 3*. Создать Canary deployment
   1. Создать два deployment'а приложения nginx.
   2. При помощи разных ConfigMap сделать две версии приложения — веб-страницы.
   3. С помощью ingress создать канареечный деплоймент, чтобы можно было часть трафика перебросить на разные версии приложения.


## Ответ:

### Описание задачи
Необходимо создать две версии приложения (nginx) с разными приветственными страницами, используя ConfigMap, и настроить Ingress-контроллер для распределения трафика между ними (Canary-деплоймент).

---

### 1. Манифест версии v1 (Main)
Файл `v1-main.yaml` содержит ConfigMap с зеленоватым фоном, Deployment и Service.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-v1-config
data:
  index.html: |
    <html>
    <body style="background-color: #e8f5e9;">
      <h1>Welcome to Version 1</h1>
      <p>This is the stable production environment.</p>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
      version: v1
  template:
    metadata:
      labels:
        app: nginx
        version: v1
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: nginx-v1-config
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-v1-service
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
    version: v1
```

### 2. Манифест версии v2 (Canary)

Файл v2-canary.yaml содержит `ConfigMap`, `Deployment` и `Service`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-v2-config
data:
  index.html: |
    <html>
    <body style="background-color: #fff3e0;">
      <h1>Welcome to Version 2 (Canary)</h1>
      <p>This is the experimental feature version.</p>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
      version: v2
  template:
    metadata:
      labels:
        app: nginx
        version: v2
    spec:
      containers:
      - name: nginx
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html-volume
        configMap:
          name: nginx-v2-config
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-v2-service
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
    version: v2
```

### 3. Манифест Ingress (Canary Logic)
Файл `ingress-canary.yaml` создает два правила. Второе правило использует аннотации для перехвата 30% трафика.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-main-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-v1-service
            port:
              number: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-canary-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "30"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-v2-service
            port:
              number: 80
```
### 4. Проверка работы
Для проверки был выполнен цикл из 10 запросов к Ingress-контроллеру (NodePort 30489) с указанием Host my-app.com.

### Команда: 

#### `for i in {1..10}; do curl -s -H "Host: my-app.com" http://192.168.1.10:30489; done`

Результат (логи): Распределение трафика составило 70% на стабильную версию (v1) и 30% на новую версию (v2), что соответствует заданному весу.










