# Программы и настройки

Для работы с кластером из под виндой

- [должна](https://ya.ru/neurum/c/tehnologii/q/chem_otlichayutsya_amdv_i_vtx_v_nastroykah_3ff40b54) быть включена виртуализация в биосе (VT-x\VMX для Intel, AMD-V\SVM для AMD)
- включен компонент виртуализации windows: Control Panel\Programs and features\Turn Windows Features on or off\Virtual Machine Platform

Установлены
- [WSL2](https://learn.microsoft.com/ru-ru/windows/wsl/install)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/#%D1%83%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-kubectl-%D0%B2-windows)
- [helm](https://github.com/helm/helm/releases/tag/v3.18.0)
- [.NET 9 SDK](https://dotnet.microsoft.com/ru-ru/download/dotnet/9.0)

- в Docker Desktop надо включить kubernetes
- kubectl тогда устанавливается автоматически
- helm должен быть доступен без прописывания абсолютного пути (можно добавить путь в PATH или положить исполняемый файл рядом с kubectl в C:/Program Files/Docker/Docker/resources/bin)

```
$ dotnet --list-sdks
9.0.xxx

$ kubectl version
Client Version: v.1.32.2

$ helm version
version.BuildInfo{Version:"v3.18.0"...
```

# [Docker](https://docs.docker.com/get-started/docker-overview/)

Docker - это платформа для контейнеризации приложений. Контейнер - это изолированная подготовленная среда для запуска приложения
- это не виртуальная машина
- любые ресурсы изолированы и могут быть лимитированы
- среда подготавливаетя путем добавления файлов в пустую папку и запуска программ
- любая команда - это слой (контрольная точка), который можно использовать потом
- после набора слоев указывается точка входа и из всего этого создается образ
- образы можно именовать тэгами и расшаривать через реестры
- команда запуска контейнера по тэгу качает образ, распаковывает на диск и проходит точку входа
- по завершению процесса точки входа контейнер останавливается

### Проверка
```
$ docker --version
Docker version 28.1.1, build 4eba3777
$ docker ps
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                    NAMES
```
### Запуск тестового контейнера
```
$ docker run --rm -it alpine:latest /bin/sh
```
### Создание приложения

```
$ dotnet new webapi --use-controllers -o Test
$ cd Test
```

Заменяем WeatherForecastController на TestController, возвращающий "hello world" по GET /test/hello
и крашаший сервер по GET /test/crash
```
using Microsoft.AspNetCore.Mvc;

namespace Test.Controllers;

[ApiController]
[Route("[controller]")]
public class TestController : ControllerBase
{
    private readonly ILogger<TestController> _logger;

    public TestController(ILogger<TestController> logger)
    {
        _logger = logger;
    }

    [HttpGet("hello")]
    public string HelloWorld()
    {
        return "hello world";
    }

    [HttpGet("crash")]
    public int Crash()
    {
        return Crash();
    }
}
```
Запускаем
```
$ dotnet run
Now listening on: http://localhost:5269
```

Проверяем
```
$ curl http://localhost:5269/test/hello
hello world

$ curl http://localhost:5269/test
404

$ curl http://localhost:5266/test/hello
unable to connect

$ curl http://localhost:5269/test/crash
500

$ curl http://localhost:5269/test/hello
unable to connect
```

### Контейнеризация

Создаем Dockerfile в корне Test
```
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /App

COPY . ./
RUN dotnet restore
RUN dotnet publish -o out

FROM mcr.microsoft.com/dotnet/aspnet:9.0
WORKDIR /App
COPY --from=build /App/out .
EXPOSE 8080
ENTRYPOINT ["dotnet", "Test.dll"]
```

Собираем образ и запускаем контейнер
```
$ docker build -t test:latest .

$ docker run test:latest
```

Проверяем что доступа нет
```
$ curl http://localhost:8080/test/hello
unable to connect
```

Перезапускаем контейнер с перенаправлением портов
```
$ docker run -p 1234:8080 test:latest
```

Проверяем
```
$ curl http://localhost:1234/test/hello
hello world
```

Список контейнеров\образов
```
$ docker ps

$ docker image ls
```

Способы убийства контейнера
```
$ docker container stop db6e55bb22a6

$ docker container exec -it 847a92f49ae3 sh
$ kill 1

$ curl http://localhost:1234/test/crash
```

Автоматический рестарт контейнера
```
$ docker run --restart always -p 1234:8080 test:latest
db6e55bb22a6

$ docker container stop db6e55bb22a6

$ docker container rm db6e55bb22a6
```

Запуск нескольких контейнеров
```
$ docker run -d -p 1234:8080 test:latest

$ docker run -d -p 1235:8080 test:latest
```
# [Kubernetes](https://kubernetes.io/)

Это система для автоматизации развертывания, масштабирования и управления
контейнеризированных приложений. Представляет из себя несколько приложений, распределенных по компьютерам,
которые следят за ресурсами и контейнерами.

Основное понятие - ресурс, задается через конфиги в yaml-формате.

## Ресурсы kubernetes

- Pod - набор контейнеров
- Deployment - набор pod'ов
- Service - настройки прокидывания портов до подов
- Ingress - настройки маршрутизации для service
- ConfigMap - общедоступные конфиги
- Secrets - зашифрованные конфиги

## Запуск kubernetes

Включаем kubernetes в настройках docker desktop.

После установки kubernetes можно посмотреть созданные по-умолчанию пространства имен и компоненты kubernetes в них, а также возможные для использования компьютеры
```
$ kubectl config use-context docker-desktop

$ kubectl get ns

$ kubectl -n kube-system get pods

$ kubectl get nodes

$ kubectl describe node docker-desktop
```

## Разворачивание приложения в kubernetes

Kubernetes надо откуда-то качать наши образы - развернем контейнер с реестром
и отправим туда образ приложения
```
$ docker run -d -p 8000:5000 --restart always --name registry registry:2

$ docker images
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
test         latest    30b001117dad   23 hours ago    330MB

$ docker tag 30b001117dad localhost:8000/test:latest

$ docker push localhost:8000/test:latest
```

### Добавим pod
pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: test
  labels:
    app: test
spec:
  containers:
  - name: test
    image: localhost:8000/test:latest
    ports:
    - containerPort: 8080
```

Проверим
```
$ kubectl apply -f pod.yaml
pod/test created

$ kubectl get pods
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          6s

$ kubectl describe pods

$ docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                    NAMES
5e5dffb8d3c7   30b001117dad   "dotnet Test.dll"        2 minutes ago    Up 2 minutes                             k8s_test_test_default_f92b7ca2-1310-4259-b851-09facb09ca17_0

$ docker container stop 5e5dffb8d3c7

$ kubectl get pods

$ kubectl delete pod test

$ kubectl exec -it test -- sh
#

$ kubectl port-forward test 1234:8080

$ curl localhost:1234/test/hello
hello world
```

### Добавим service
service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: test
spec:
  type: NodePort
  selector:
    app: test
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 32323
```

Проверяем
```
$ kubectl apply -f ./service.yaml
service/test configured

$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
test         NodePort    10.101.239.132   <none>        8080:32323/TCP   28m

$ curl localhost:32323/test/hello
hello world
```

## Deployment
deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
  labels:
    app: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: test
        image: localhost:8000/test:latest
        ports:
        - containerPort: 8080
```

Проверим
```
$ kubectl apply -f ./deployment.yaml
deployment.apps/test created

$ kubectl get deploy
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   1/1     1            1           6s

$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
test-6674b854c-9bjx8   1/1     Running   0          9s

$ kubectl edit deploy test
deployment.apps/test edited

$ kubectl get pods
NAME                   READY   STATUS    RESTARTS   AGE
test-6674b854c-9bjx8   1/1     Running   0          112s
test-6674b854c-ckgb9   1/1     Running   0          24s

$ kubectl rollout restart deployment test

$ kubectl delete deploy test

$ kubectl delete svc test
```

# [Helm](https://helm.sh)

Менеджер пакетов для kubernetes - предоставляет доступ к наборам yaml ресурсов kubernetes через репозитории, поддерживает шаблонизацию конфигов. Наборы принято называть chart

## Чарт нашего сервиса

```
$ helm create helm
Creating helm

$ rm -rf helm/charts
$ rm -rf helm/templates/tests
$ rm helm/templates/hpa.yaml
$ rm helm/templates/NOTES.txt
$ rm helm/templates/serviceaccount.yaml
$ rm helm/.helmignore
```

Правим helm/values.yaml
```
replicaCount: 2

image:
  repository: localhost:8000/test
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 8080

autoscaling:
  enabled: false
serviceAccount:
  create: false

ingress:
  enabled: true
  className: "nginx"
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: kubernetes.docker.internal
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

livenessProbe:
  httpGet:
    path: /test/hello
    port: http
readinessProbe:
  httpGet:
    path: /test/hello
    port: http
```

## Установка чарта
```
$ helm install test ./helm -f ./helm/values.yaml
NAME: test

$ kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
test-helm-68b78c595-nzhtn   1/1     Running   0          49s
test-helm-68b78c595-xlgtv   1/1     Running   0          47s
```

## Установка nginx как ingress-контроллера
```
$ helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace --set controller.progressDeadlineSeconds=null

$ kubectl -n ingress-nginx get pods
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-6885cfc548-64n82   1/1     Running   0          4h14m

$ kubectl get ing
NAME        CLASS   HOSTS                        ADDRESS     PORTS   AGE
test-helm   nginx   kubernetes.docker.internal   localhost   80      17m

$ kubernetes.docker.internal/test/hello
bash: kubernetes.docker.internal/test/hello: No such file or directory

$ curl kubernetes.docker.internal/test/hello
hello world

```

## Crash-tolerance и логи
```
$ curl kubernetes.docker.internal/test/crash
<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx</center>
</body>
</html>

$ kubectl get pods
NAME                        READY   STATUS    RESTARTS     AGE
test-helm-68b78c595-nzhtn   1/1     Running   1 (3s ago)   4h32m
test-helm-68b78c595-xlgtv   1/1     Running   1 (8s ago)   4h32m

$ kubectl logs test-helm-68b78c595-nzhtn
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://[::]:8080
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /App
warn: Microsoft.AspNetCore.HttpsPolicy.HttpsRedirectionMiddleware[3]
      Failed to determine the https port for redirect.

$ kubectl logs -p test-helm-68b78c595-nzhtn
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://[::]:8080
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Production
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /App
warn: Microsoft.AspNetCore.HttpsPolicy.HttpsRedirectionMiddleware[3]
      Failed to determine the https port for redirect.
Stack overflow.
Repeated 261809 times:
--------------------------------
   at Test.Controllers.TestController.Crash()
--------------------------------
   at DynamicClass.lambda_method3(System.Runtime.CompilerServices.Closure, System.Object, System.Object[])
```

## Удаление чарта
```
$ helm ls
NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
test    default         2               2025-05-22 02:48:57.8301276 -0700 PDT   deployed        helm-0.1.0      1.16.0

$ helm uninstall test
release "test" uninstalled

$ helm -n ingress-nginx ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
ingress-nginx   ingress-nginx   2               2025-05-22 03:04:09.490059 -0700 PDT    deployed        ingress-nginx-4.12.2    1.12.2

$ helm -n ingress-nginx uninstall ingress-nginx
release "ingress-nginx" uninstalled
```
