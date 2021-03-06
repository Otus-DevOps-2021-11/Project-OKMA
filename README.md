# Project-OKMA
## План курсовой:

1. Создать развертывание кубер-кластера на 2 ноды 4/8/64 в `terraform`
2. Написать Докер-файлы для развёртывания приложения в докере
3. Залить докер-образы в докер-хаб
4. Написать деплойменты приложения в кубер-кластер
5. Установить nginx ingress-controller в кластер:

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml`

6. Написать правило `ingress` для приложения
7. Деплоим приложение и `ingress`
8. Запилить логгирование
9. Запилить мониторинг и алерты
10. Смешная 11 опция...

### Приложения:
[CRAWLER](https://github.com/express42/search_engine_crawler)

[UI](https://github.com/express42/search_engine_ui)

### Разворот managed k8s кластера

Приложение развёрнуто в kubernetes кластере. Кластер состоит из двух нод, кластер создаётся через Terraform. Для
создания kubernetes-кластера нужно:

1. Перейти в каталог `terraform`
2. В файлах `variables.tf` и `terraform.tfvars` заменить значения всех переменных (кроме переменной `instance count`).
3. Выполнить команду `terraform init`
4. Выполнить команду `terraform apply`
5. После создания кластера чтобы к нему подключиться в консоли Host'a выполнить команду 
   `yc managed-kubernetes cluster get-credentials okmacluster --external --force`
6. Для создания неймспейса перейти в каталог `kubernetes/app` и выполнить команду `kubectl apply -f namespace.yml`

### Docker образы

* Для работы приложения в `dockerfil'ах` изменили версию python на 3.9
* изменили зависимости приложения (файлы`requirements.txt`): обновили используемую версию `flask` до 2.0.3
* для микросервисов `crawler` и `crawler_ui` были собраны Docker-образы, образы находятся в каталогах `search_crawler`  и `search_crawler_ui` для    `crawler` и `crawler_ui` соответственно.
* Образы добавили в репозиторий Docker-hub. Для работы `crawler_ui` в `requirements.txt` добавили также библиотеку `markupsafe`.

## Kubernetes

### Установка nginx ingress-controller
Выполните команду

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml`


### Deploy Rabbitmq

**Важно!** В случае, если в файле `docker/search_crawler/Dockerfile` в качестве значения переменных `ENV RMQ_USERNAME`,
`ENV RMQ_PASSWORD` не используется`rabbitmq`, то в файле `kubernetes/rmq/rabbitmq_statefulset.yaml` в разделе `env`
(ориентировочно строки 83 - 86) нужно заменить значения переменных `RABBITMQ_DEFAULT_USER` и `RABBITMQ_DEFAULT_PASS` на
значения этих переменных из файла `docker/search_crawler/Dockerfile`.

1. Перейдите в каталог `kubernetes`

2. Выполните команду `kubectl apply -f namespace.yml`

3. Перейдите в каталог `kubernetes\rmq`

4. Последовательно выполните команды

  * `kubectl apply -f rabbitmq_rbac.yaml -n dev`
  * `kubectl apply -f rabbitmq_pv.yaml -n dev`
  * `kubectl apply -f rabbitmq_pvc.yaml -n dev`
  * `kubectl apply -f rabbitmq_service.yaml -n dev`
  * `kubectl apply -f rabbitmq_service_ext.yaml -n dev`
  * `kubectl apply -f rabbitmq_configmap.yaml -n dev`
  * `kubectl apply -f rabbitmq_statefulset.yaml -n dev`

### Деплой приложения в kubernetes-кластер

1. Создайте диск для хранения БД, для этого выполните команду

  `yc compute disk create k8s --size 4 --description "disk for okmadb"`,

2. Командой `yc compute disk list` получаем ID созданного диска

3. Полученный ID нужно указать в разделе `volumeHandle` файле `kubernetes/app/mongo-volume.yml`

4. В терминале перейдите в каталог `kubernetes/app`

5. Последовательно выполните следующие команды:

  `kubectl apply -f mongo-volume.yml -n dev`

  `kubectl apply -f mongo-claim.yml -n dev`

  `kubectl apply -f mongo-deployment.yml -n dev`

  `kubectl apply -f mongodb-service.yml -n dev`

  `kubectl apply -f crawler-ui-deployment.yml -n dev`

  `kubectl apply -f crawler-ui-service.yml -n dev`

  `kubectl apply -f crawler-ui-mongodb-service.yml -n dev`

  `kubectl apply -f crawler1-deployment.yml -n dev`

  `kubectl apply -f crawler1-mongodb-service.yml -n dev`

  `kubectl apply -f crawler1-rabbitmq-service.yml -n dev`


## Мониторинг
#### Используемый стек:

**Prometheus+Alertmanager**

#### Docker images
 * Решены проблемы с совместимостью плагина для `fluentd` и версии `elasticsearch`
 * решили проблему с падением `elasticsearch` из-за нехватки памяти
 * для запуска мониторинга собран `docker-compose` в файле `docker-monitoring.yml`
 * Добавлен файл kube-prometheus-stack.yml с настройками для разворота через helm
 * Чтобы развернуть kube-prometheus-stack выполняем:
 `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
 
 `helm repo update`
 
`helm install --set name=dev monitoring prometheus-community/kube-prometheus-stack -f monitoring/kube-prometheus-stack.yml`

#### Alerting

  * Создано правило отправки `alert'ов` при падении сервиса (если `up` одного из `endpoint == 0`)
  * для получения алертов создан канал `monitoring-okma` в Slack. В канал добавлены все участники проекта
  * Оповещение при падении сервиса приходит.


#### Gitlab-CI
Непрерывное развертывание контейнеризованных приложений с помощью GitLab в Яндексе:

`https://cloud.yandex.ru/docs/tutorials/infrastructure-management/gitlab-containers`

`helm repo add gitlab-ci https://charts.gitlab.io`
Установить раннер в кубер:
`helm install --namespace default gitlab-runner -f gitlab-values.yaml gitlab/gitlab-runner`
Убедитесь, что под GitLab Runner перешел в состояние Running:

`kubectl get pods -n default | grep gitlab-runner`
`gitlab-runner-gitlab-runner-6d5f667499-l4jtg   1/1     Running   0  64s`


#### Деплоим Loki\Grafana
Для установки выполните команды:
1. `kubectl create ns monitoring`
2. `helm repo add grafana https://grafana.github.io/helm-charts`
3. `helm repo update`
4. `helm upgrade --install loki --namespace=monitoring grafana/loki-stack`
5. `helm upgrade --install grafana --namespace=monitoring grafana/grafana`
6. `kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo`
7. `kubectl apply -f kubernetes/grafana/grafana_ingres.yaml -n monitoring --force`

#### Ingress
Для доступа к сервисам используйте следующие адреса:

0. `https://gitlab.maxx.su/otus/project-okma` - репозиторий проекта
1. `http://crawler.maxx.su` - приложение
2. `http://rabbit.maxx.su` - rabbitmq
3. `http://alertmanager.maxx.su` - alertmanager
4. `http://prometheus.maxx.su` - prometheus
5. `http://grafana.maxx.su` - grafana (здесь собраны логи и метрики)
