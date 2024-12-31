# Дипломная работа по направлению "DevOps-инженер с нуля"

* [ТЗ](<Terms of reference.md>)

## Все созданные составляющие итоговой системы

* Модули terraform:
    * [simple-vpc](https://github.com/RedRatInTheHat/simple-vpc)
    * [simple-vms](https://github.com/RedRatInTheHat/simple-vms)
    * [simple-vmg](https://github.com/RedRatInTheHat/simple-vmg)
    * [simple-ab](https://github.com/RedRatInTheHat/simple-ab)
* Сервисный аккаунт:
    * [terraformer](https://github.com/RedRatInTheHat/terraformer)
* Bucket для tfstate:
    * [terraform-bucket](https://github.com/RedRatInTheHat/terraform-backend)
* Инфраструктура для kubernetes:
    * [terraform-for-k8s](https://github.com/RedRatInTheHat/terraform-for-k8s)
* Контейнер со статическим сайтом:
    * [static-site](https://github.com/RedRatInTheHat/static-site)
    * [DockerHub](https://hub.docker.com/repository/docker/redratinthehat/static-mark/general)

## Решение

### Terraform

#### Backend

Для работы с инфраструктурой ранее создавался сервисный аккаунт `terraform`, так что в коде [terraformer](https://github.com/RedRatInTheHat/terraformer) он импортируется. Ему предоставлены доступы editor'а и storage.admin'а для создания ресурсов в отдельной папке diploma и взаимодействия с объектами bucket'а.

Добавлено создание S3 bucket'а с помощью Terraform; код содержится в репозитории [terraform-backend](https://github.com/RedRatInTheHat/terraform-backend).

В проект для создания инфраструктуры в Terraform подлкючен функционал сохранения файла состояний в S3 bucket'е: [main.tf](https://github.com/RedRatInTheHat/terraform-for-k8s/blob/master/main.tf)

Файл состояний успешно сохраняется в bucket'е:

![alt text](img/1.2.png)

#### Инфраструктура для K8S

* Для создания сети использован ранее написанный модуль [simple-vpc](https://github.com/RedRatInTheHat/simple-vpc). Подключение модуля: [vpc.tf](https://github.com/RedRatInTheHat/terraform-for-k8s/blob/master/vpc.tf).
* Для создания bastion host и master node использован ранее написанный модуль [simple-vms](https://github.com/RedRatInTheHat/simple-vms). Подключение модуля: [vms.tf](https://github.com/RedRatInTheHat/terraform-for-k8s/blob/master/vms.tf).<br/>
* Для создания worker node использован модуль [simple-vmg](https://github.com/RedRatInTheHat/simple-vmg), который создаёт группу инстансов. Модуль подключается там же, в vms.tf.
* Для обеспечения доступа к созданным машинам, создан Application Load Balancer. Его создание вынесено в модуль [simple-ab](https://github.com/RedRatInTheHat/simple-ab), а сам модуль подключается в [application-load-balancer.tf](https://github.com/RedRatInTheHat/terraform-for-k8s/blob/master/application-load-balancer.tf)

Как это в итоге работает:
1. Создаются master-ноды (отдельные виртуальные машины) и worker-ноды в составе группы виртуальных машин в заданном количестве.<br/>
У них нет публичных адресов, только внутренние.<br/>
Для обеспечения отказоустойчивости master тоже стоило бы создавать в составе группы, но они созаются дольше и обрастают неожиданными ошибками (*нет, я не могу удалить публичный адрес у одной единственной машины, а теперь я не могу удалить машины вообще*), так что в составе ученической работы был оставлен такой вариант.

2. ip адреса машин записываются в файл inventory, который позднее используется при настройке k8s с помощью kubespray.

3. Для доступа к виртуальным машинам для Ansible используется ProxyJump в файле `.ssh/config`, а для доступа пользователей к приложению, поднятому в k8s, настроен load balancer.<br/>
Для работы с удалённым master-узлом используется SSH-туннелирование.<br/>
Доступ пользователей к приложениям производится через публичный адрес роутера, а для health check настроено добавление название host'а в header'e.

4. Группы безопасности не настроены, а зря.

#TODO настроить группы безопасности
#TODO настроить скриншоты для отображения того, что всё ок и поднялось.

### Ansible

#### Kubespray

С помощью Kubespray производится установка k8s на ноды, созданные Terraform. Как уже упоминалось, доступ к нодам производится с помощью ProxyJump.

В файлах kubespray внесены следующие изменения:
* файл `inventory/cluster/inventory.yaml` создаётся автоматически после создания виртуальных машин.
* в файле `inventory/cluster/group_vars/k8s_cluster/addons.yml` подключен Nginx Ingress Controller (Раздел `# Nginx ingress controller deployment`).

### Docker

Для создания docker-контейнера со статическим сайтом, добавлен репозиторий [static-site](https://github.com/RedRatInTheHat/static-site). 

В репозитории расположены конфигурация nginx, директория html со страницей в ней, и Dockerfile с инструкциями сборки контейнера.

Контейнер выложен в [DockerHub](https://hub.docker.com/repository/docker/redratinthehat/static-mark/general).

### Kubernetes

В K8S поднимаются следующие составляющие:
*  с выше описанным приложением, [deployment-ca.yml](https://github.com/RedRatInTheHat/diploma-k8s/blob/master/deployment-ca.yml);
* сервис для этого приложения, [service-ca.yml](https://github.com/RedRatInTheHat/diploma-k8s/blob/master/service-ca.yml);
* grafana-prometheus-node-exporter – запускается проект [kube-prometheus](https://github.com/prometheus-operator/kube-prometheus). Запускаются только apply описанным в нём способом, дополнительных настроек не вносится;
* дополнительно настроиваются сетевые политики для подов Grafana, [grafana-network-policy.yml](https://github.com/RedRatInTheHat/diploma-k8s/blob/master/grafana-network-policy.yml), так как по умолчанию они доступны только внутренним подам, а мы стучимся с nginx-контроллера;
* ingress для своего приложения, [ingress-ca.yml](https://github.com/RedRatInTheHat/diploma-k8s/blob/master/ingress-ca.yml);
* ingress для grafana, [ingress-monitoring.yml](https://github.com/RedRatInTheHat/diploma-k8s/blob/master/ingress-monitoring.yml).

### Atlantis

По описанию как будто должен располагаться в K8S кластере. И всё же было решено создать отдельную машину с Atlantis (причём в обычном docker-контейнере). А иначе получается, что сервис в K8S перестраивает сам себя.

Поднятие приложения было реализовано через:
* поднятие виртуальной отдельной виртуальной машины через Terraform: [vm.tf](https://github.com/RedRatInTheHat/diploma-atlantis/blob/master/vm.tf)
* поднятие контейнера с Atlantis, куда так или иначе закинуты все нужные файлы конфигураций. Бизнес-процесс default заменён для возможности указывать файл переменных и токены инициализации. Всё это разворачивается через [cloud-init.yml](https://github.com/RedRatInTheHat/diploma-atlantis/blob/master/cloud-init.yml).
* отслеживаются pull-requests; после планирования требуется комментарием подтвердить внесение изменений.

Что следовало бы изменить:
* разворачивать на виртуальной машине необходимые структуры через Ansible.
* удалять файлы конфигуаций сразу после поднятия контейнеров.

### CI/CD

В проекте static-site настроен workflow для автоматического применения изменений в директории, [static-deploy.yml](https://github.com/RedRatInTheHat/static-site/blob/master/.github/workflows/static-deploy.yml).

Образ собирается и отправляется в Docker Hub; в кластере K8S для Deployment'а применяется новый образ. Единственная загвоздка в создании SSH тоннеля – для этого приходится править конфигурации и, собственно, создавать тоннель через bastion host.
