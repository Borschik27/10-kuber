# Подготовка:
## Установка helm в систему:

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

helm version
version.BuildInfo{Version:"v3.17.0", GitCommit:"301108edc7ac2a8ba79e4ebf5701b0b6ce6a31e4", GitTreeState:"clean", GoVersion:"go1.23.4"}
```

## Создадим свой heml-deploy:
```
helm create bysybox-multitool
```

# Описание:

## После создания получаем такую структуру

---
```plaintext
busybox-multitool/
├── charts
├── Chart.yaml
├── templates/
│   ├── deployment.yaml        # Шаблон для Deployment
│   ├── _helpers.tpl           # Переиспользуемые шаблоны
│   ├── hpa.yaml               # Шаблон для HorizontalPodAutoscaler
│   ├── ingress.yaml           # Шаблон для Ingress
│   ├── NOTES.txt              # Инструкции для пользователя после установки
│   ├── serviceaccount.yaml    # Шаблон для ServiceAccount
│   ├── service.yaml           # Шаблон для Service
│   └── tests/
│       └── test-connection.yaml  # Тесты для проверки подключения
├── values-dev.yaml            # Значения для разработки, добавлен позднее самостоятельно, его нет при создании
├── values-prod.yaml           # Значения для продакшн, добавлен позднее самостоятельно, его нет при создании
└── values.yaml                # Базовые значения
---
```

# Разберем ее

---
Основные файлы которые требуются для работы, остальные файлы могу быть удалены:

  1. Chart.yaml — обязательный файл для описания самого чарта.
  2. templates/deployment.yaml — описание подов и логики приложения.
  3. templates/service.yaml — для экспонирования приложения через сервис.
  4. values.yaml — базовые значения, которые подставляются в манифесты.
---

# Описание файлов:
##  0. **`charts/`**
  Используется для организации зависимостей. Если ваше приложение зависит от других чартов, то эти зависимости будут находиться в каталоге charts/.

  1. Зависимости: Когда ваше приложение зависит от других чарта, вы можете добавить эти чарты в каталог charts/. Это позволяет вам инкапсулировать другие чарты и их зависимости прямо в вашем проекте.
  2. Подкаталог: Чарты, добавленные в этот каталог, могут быть скачаны из других репозиториев, или это могут быть локальные чарты, которые включаются непосредственно в ваш проект.

---
```plaintext
  busybox-multitool/
  ├── charts/
  │   ├── nginx/
  │   └── postgresql/
```
---

##  1. **`Chart.yaml`**
  Этот файл содержит метаданные о чарте. Он необходим для описания версии, названия и описания чарта.

## 2. **`templates/`**
  Это директория, в которой находятся шаблоны манифестов Kubernetes, генерируемые Helm. Манифесты создаются на основе значений, переданных в `values.yaml`.

### 2.1 **`deployment.yaml`**
  Шаблон для создания объекта `Deployment`, который отвечает за развертывание приложения в Kubernetes. В нем указаны контейнеры, образы, команды и другие параметры.

### 2.2 **`_helpers.tpl`**
  Этот файл содержит переиспользуемые шаблоны и функции. Он позволяет создавать общие элементы, такие как имена ресурсов.

### 2.3 **`hpa.yaml`**
  Шаблон для создания объекта **Horizontal Pod Autoscaler** (HPA), который автоматически масштабирует количество реплик в зависимости от использования ресурсов.

### 2.4 **`ingress.yaml`**
  Шаблон для создания объекта **Ingress**, который управляет внешним доступом к приложениям, работающим в Kubernetes. Используется для настройки маршрутизации HTTP(S) запросов.

### 2.5 **`NOTES.txt`**
  Этот файл выводится пользователю после успешного деплоя. Здесь могут быть инструкции или подтверждения о статусе деплоя.

### 2.6 **`serviceaccount.yaml`**
  Шаблон для создания **ServiceAccount** — учетной записи, которая используется для выполнения операций в кластере.

### 2.7 **`service.yaml`**
  Шаблон для создания объекта **Service**, который позволяет другим приложениям или пользователям взаимодействовать с вашим приложением.

##  3. **`tests/test-connection.yaml`**
  Этот файл содержит тесты, которые выполняются после установки чарта. Они проверяют, доступно ли ваше приложение или сервис.

##  4. **`values.yaml`**
  Эти файлы содержат значения, которые подставляются в шаблоны для конфигурации чарта. Они позволяют настроить приложение для разных окружений.


# Задача 1

## Структура helm:
---
```plaintext
.
└── busybox-multitool
    ├── Chart.yaml
    ├── templates
    │   ├── deployment.yaml
    │   └── service.yaml
    ├── values-dev.yaml
    ├── values-prod.yaml
    └── values.yaml
```
---

## Манифесты:
### deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-busybox-multitool
  labels:
    app: {{ .Chart.Name }}
    release: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      volumes:
      - name: shared-data
        emptyDir: {}
      containers:
      - name: busybox
        image: {{ .Values.busybox.image }}
        volumeMounts:
        - name: shared-data
          mountPath: /data
        command:
        - sh
        - -c
        - "while true; do echo $(date) >> /data/shared.log; sleep 20; done"
      - name: multitool
        image: {{ .Values.multitool.image }}
        volumeMounts:
        - name: shared-data
          mountPath: /data
        command:
        - sh
        - -c
        - "while true; do cat /data/shared.log; sleep 20; done"
```

### service
```
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-busybox-service
  labels:
    app: {{ .Chart.Name }}
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: {{ .Chart.Name }}
```

### values/valuse-dev
```
replicaCount: 1

busybox:
  image: busybox:latest

multitool:
  image: praqma/network-multitool:latest
```

### values-prod
```
replicaCount: 3

busybox:
  image: busybox:stable

multitool:
  image: praqma/network-multitool:stable
```

## Запусти наш helm-chart
  1. Запускается с параметрами взятыми из values.yaml
     ![image](https://github.com/user-attachments/assets/2ed89798-7c84-4f53-9ad4-869ed067aeef)
     ![image](https://github.com/user-attachments/assets/88c658fd-3c9b-4382-a604-47b8289e1da6)

  2. Запустим с параметрами values-prod.yaml
     ![image](https://github.com/user-attachments/assets/00d4ca35-3195-45d9-8066-806fc0442b60)
     ![image](https://github.com/user-attachments/assets/9ee72eda-2473-4b0a-bf1a-9a362cea4945)
    
  4. Количество реплик можно изменить следующим образом
     ```
      helm upgrade busybox-multitool ./busybox-multitool --set replicaCount=3 --set image.tag=v2.0
     ```
     ![image](https://github.com/user-attachments/assets/4f023878-f52c-4a93-83cd-77b523738627)
     ![image](https://github.com/user-attachments/assets/a2887699-e334-4243-aeb2-3b3d2b8a5786)

  5. Удалить helm
     ```
     helm uninstall <name> -n <name-space>
     ```
     ![image](https://github.com/user-attachments/assets/0fa98d70-0ead-4509-970f-4a01f8a835ad)

