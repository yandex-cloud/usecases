# CI/CD в Облаке с помощью GitLab

В практической части вебинара мы создаем блог с использованием фреймворка [HUGO](https://gohugo.io/) 
для генерации статического содержания. На GitLab опираемся как на основной элемент CI/CD инфрастуруктуры.  
Мы используем Docker как контейнер и для хранения Docker-образов используем Container Registry, 
всю инфраструктуру разворачиваем в кластере Kubernetes.

## Видео вебинара

## Дополнительные источники
1. Актуальная документация по интеграции GitLab и Kubernetes доступна в документации по адресу:
[Непрерывное развертывание контейнеризованных приложений с помощью GitLab](https://cloud.yandex.ru/docs/solutions/infrastructure-management/gitlab-containers)
2. Для базовой настройки блога с применением HUGO можно воспользоваться инструкцией: 
[Quick Start](https://gohugo.io/getting-started/quick-start/)
3. Статья Florent Chauveau: 
[Dockerized Hugo with GitLab CI/CD](https://blog.callr.tech/static-blog-hugo-docker-gitlab/)
4. Пример репозитория с блогом в финальной версии:
[webinar_cicd_hugoblog](https://github.com/golodnyj/webinar_cicd_hugoblog)

## Последовательность действий
### Создайте виртуальную машину blog
### Обновите blog и установим дополнительное ПО:
`sudo apt-get update`    
`sudo apt-get install build-essential curl file git`  
`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"`  
`brew install gcc`  
`brew install kubectl`  
`curl https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash`  

### Установите и настроим HUGO
Действуем согласно инструкции [Quick Start](https://gohugo.io/getting-started/quick-start/).   
`brew install hugo`  
`hugo new site blog`  
`cd blog`  
`git init`  
`git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke`  
`echo 'theme = "ananke"' >> config.toml`  
`hugo server`  

### Создайте виртуальную машину GitLab
Используем образ виртуальной машины уже с установленным GitLab. В сервисе Virtual Private Cloud сделаем публичный адрес виртуальной машины GitLab статичным. Зададим пароль root пользователя в GitLab. Создадим отждельного пользователя с правами Admin. Создадим проект `blog`, с виртуальной машины blog, можно выложить содержание папки статического сайта в репозиторий.    
### Создайте реестр Container Registry
После создания реестра запишите идентификатор, он потребуется далее.

### Создайте сервисные аккаунты
Можно сделать один аккаунт, но лучше два. Один с ролью `editor`, второй с ролью `container-registry.images.pusher`. Аккаунты потребуются для создания кластера Kubernetes

### Создайте кластер Kubernetes
Используйте созданные ранее сервисные аккаунты. Сохраните идентификатор кластера — он понадобится для следующих шагов. После создания кластера Kubernetes создайте в нем группу узлов.

### Получите данные для интеграции Kubernetes с GitLab
В консоле на виртуальной машине blog, установим утилиту jq `sudo apt-get install jq`. Инициализируйте консоль `yc` с помощью команды `yc init`, по полученному адресу получите OAuth токен. Токен храните в секрете, он также потребуется позже. Настройте локальное окружение на работу с созданным кластером Kubernetes:  
`yc managed-kubernetes cluster get-credentials <cluster-id> --external`  

Сохраните спецификацию для создания сервисного аккаунта Kubernetes в YAML-файл `gitlab-admin-service-account.yaml`:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gitlab-admin
  namespace: kube-system
```

Создайте сервисный аккаунт и получите его секрет:  
`kubectl apply -f gitlab-admin-service-account.yaml`    
`kubectl -n kube-system get secrets -o json | \
jq -r '.items[] | select(.metadata.name | startswith("gitlab-admin")) | .data.token' | \
base64 --decode`  

Получите адрес узла мастера Kubernetes кластера:  
`yc managed-kubernetes cluster get <cluster-id> --format=json \
| jq -r .master.endpoints.external_v4_endpoint`

Получите сертификат мастера Kubernetes кластера CA Certificate:
`yc managed-kubernetes cluster get <cluster-id> --format=json \
| jq -r .master.master_auth.cluster_ca_certificate`   

### Подключите кластер Kubernetes к сборкам GitLab.  
### Установите на кластер Kubernetes приложения Helm Tiller и GitLab Runner.  
### В настройках CI/CD создайте переменные:
Задайте переменные `KUBE_URL`, `KUBE_TOKEN`, `OAUTH`, `REGISTRYID` — они потребуются при создании файла `.gitlab-ci.yml`


### Настройте CI/CD
Создайте файл `Dockerfile`
```
# This is a multi-stage Dockerfile (build and run)

# Remember to target specific version in your base image,
# because you want reproducibility (in a few years you will thank me)
FROM alpine:3.9 AS build

# The Hugo version
ARG VERSION=0.70.0

ADD https://github.com/gohugoio/hugo/releases/download/v${VERSION}/hugo_${VERSION}_Linux-64bit.tar.gz /hugo.tar.gz
RUN tar -zxvf hugo.tar.gz
RUN /hugo version

# We add git to the build stage, because Hugo needs it with --enableGitInfo
RUN apk add --no-cache git

# The source files are copied to /site
COPY . /site
WORKDIR /site


# And then we just run Hugo
RUN /hugo --minify --enableGitInfo

# stage 2
FROM nginx:1.15-alpine

WORKDIR /usr/share/nginx/html/

# Clean the default public folder
RUN rm -fr * .??*

# This inserts a line in the default config file, including our file "expires.inc"
RUN sed -i '9i\        include /etc/nginx/conf.d/expires.inc;\n' /etc/nginx/conf.d/default.conf

# The file "expires.inc" is copied into the image
COPY expires.inc /etc/nginx/conf.d/expires.inc
RUN chmod 0644 /etc/nginx/conf.d/expires.inc

# Finally, the "public" folder generated by Hugo in the previous stage
# is copied into the public fold of nginx
COPY --from=build /site/public /usr/share/nginx/html
```

Создайте файл `k8s.yaml`
```
apiVersion: v1
kind: Namespace
metadata:
  name: blog-world
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog-deployment
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog
  template:
    metadata:
      namespace: default
      labels:
        app: blog
    spec:
      containers:
        - name: blog-container
          image: cr.yandex/crp39ijmhh8q955lms3v/blog:__VERSION__
          imagePullPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  namespace: default
  name: my-nginx
  labels:
    app: blog
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: blog
  type: LoadBalancer
```

Создайте файл `.gitlab-ci.yml`
```
stages:
  - build
  - deploy

build:
  stage: build
  image: docker:18-git 
  
  variables:
   DOCKER_DRIVER: overlay2
   DOCKER_TLS_CERTDIR: ""
   DOCKER_HOST: tcp://localhost:2375/
  
  services:
    - docker:19.03.1-dind

  only:
    - master
    
  before_script:
    - git submodule sync --recursive
    - git submodule update --init --recursive
  
  script:
    - docker login -u oauth -p $OAUTH cr.yandex
    - docker build -t cr.yandex/$REGISTRYID/blog:$CI_COMMIT_SHORT_SHA .
    - docker push cr.yandex/$REGISTRYID/blog:$CI_COMMIT_SHORT_SHA

deploy:
  image: gcr.io/cloud-builders/kubectl:latest
  stage: deploy
  script:
    - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
    - kubectl config set-credentials admin --token="$KUBE_TOKEN"
    - kubectl config set-context default --cluster=k8s --user=admin
    - kubectl config use-context default
    - sed -i "s/__VERSION__/$CI_COMMIT_SHORT_SHA/" k8s.yaml
    - kubectl apply -f k8s.yaml
```
Каждый следующий комит в репозиторий будет запускать цикл инструкций описанных в  файле `.gitlab-ci.yml`
