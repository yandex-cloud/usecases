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
1. Создать виртуальную машину blog
1. Создать виртуальную машину gitlab
1. Создать Container Registry
1. Создать кластер kubernetes
1. Добавить интеграцию в gitlab с kubernetes
1. Инсталировать GitLab Runner
1. Настроить CI/CD
