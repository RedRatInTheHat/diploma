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
* Atlantis:
    * [diploma-atlantis](https://github.com/RedRatInTheHat/diploma-atlantis)

## Решение

### Terraform

#### Backend

Для работы с инфраструктурой ранее создавался сервисный аккаунт `terraform`, так что в коде [terraformer](https://github.com/RedRatInTheHat/terraformer) он импортируется. Ему предоставлены доступы editor'а и storage.admin'а для создания ресурсов в отдельной папке diploma и взаимодействия с объектами bucket'а.

Добавлено создание S3 bucket'а с помощью Terraform; код содержится в репозитории [terraform-backend](https://github.com/RedRatInTheHat/terraform-backend).

В проект для создания инфраструктуры в Terraform подлкючен функционал сохранения файла состояний в S3 bucket'е: [main.tf](https://github.com/RedRatInTheHat/terraform-for-k8s/blob/master/main.tf).

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

## Как это всё разворачивается

Как говорила моя преподавательница по линалу, с болью.

### Кластер K8S

1. В проекте terraform-for-k8s запускается `terraform apply`.
2. IP созданного бастиона добавляется:
    * В файл конфигурации `~/.ssh/config` заносится запись вида:
    ```
    Host bastion
        Hostname <bastion_ip>
        User <username>

    Host 192.168.*
        ProxyJump bastion
        User <username>
    ```
    * В переменную BASTION_IP для проекта `static-site` (GitHub).
3. Сложить внутренний ip master'а в переменную `K8S_SERVER` для проекта `static-site` (GitHub).
3. Запустить `kubespray`. В моей структуре файл `invetnory.yaml` складывается сразу в нужную директорию; в противном случае его нужно нести руками.<br/>
Итак, запускается `ansible-playbook -i inventory/cluster/inventory.yaml cluster.yml -b`.<br/>
Ждётся.
4. Забрать config из master'а и сложить:
    * в `.kube/config` на рабочей машине
    * в переменную KUBE_CONFIG для проекта `static-site` (GitHub).
5. Настроить тоннель на рабочей машине: `ssh -fN -L 6443:<master_ip>:6443 ubuer@<bastion_ip>` (периодически убивать и создавать снова, потому что зависает, а добавить в cloud-init редактирование файла `sshd_config` всё руки не доходят).
7. Установить `kube-prometheus` – тут без неожиданностей, всё устанавливается командами, рекомендованными разработчиками.
```
kubectl apply --server-side -f manifests/setup
kubectl wait --for condition=Established --all CustomResourceDefinition --namespace=monitoring
kubectl apply -f manifests/
``` 
8. Подключить свои элементы K8S `kubectl apply -f k8s/`.
9. Дождаться, пока выздоровеет балансировщик.
10. Обновить `/etc/hosts`:
```
<ip_балансировщика> static.redrat.diploma
<ip_балансировщика> grafana.redrat.diploma
```
11. Создать пользователя в Grafana.<br/>
Доски там как будто и без того хороши, так что дополнительных не добавлено.

Отдельно ото всего поднимается Atlantis:
1. В директории `diploma-atlantis` запустить Terraform.
2. IP поднятой машины закинуть в GitHub Webhooks проекта `terraform-for-k8s`.

## Итог

### Что сделано

Сервисному аккаунту выданы права editor'а и storage.admin'а, и этого ему достаточно:

![alt text](img/1.1.png)

В Yandex Cloud созданы: 
* 5 виртуальных машин (1 master, 2 worker'а, 1 bastion, 1 atlantis);
* Load Balancer (со сопутствующими ему роутером, целевой группой и группой бэкендов);
* 2 сети (так как atlantis поднимается отдельно и импортировать ему сеть придётся вручную, а это не весело);
* Object Storage (для файлов состояний Terraform).

![alt text](img/1.2.png)

![alt text](img/1.3.png)

![alt text](img/1.4.png)

Создан и залит образ приложения, выдающего статическую страницу:

![alt text](img/1.5.png)

В кластере развёрнуты приложение и стек grafana-prometheus-node-exporter:

![alt text](img/1.6.png)

![alt text](img/1.7.png)

![alt text](img/1.8.png)

Приложения доступны:

![alt text](img/1.9.png)

![alt text](img/1.10.png)

Настроен Atlantis; при pull-request'е он запускает plan и, при вводе команды, готов его применять:

![alt text](img/1.11.png)

https://github.com/RedRatInTheHat/terraform-for-k8s/pull/4

![alt text](img/1.12.png)

Настроен CI/CD для приложения со статической страницей:

https://github.com/RedRatInTheHat/static-site/actions

![alt text](img/1.13.png)

![alt text](img/1.14.png)

### Что стоило бы сделать (но Новый год)

1. Создать группы безопасности – а-то у нас две машины наружу торчат, другие внутрь попрятали, а групп безопасности нет, осуждаемо.
2. Использовать Ansible вместо Cloud-init.
3. Собрать всё в единый процесс. Сейчас много этапов, как видно из раздела "Как это всё разворачивается", производится вручную, при том, что локальные файлы точно можно поправить скриптами, а для других процессов наверняка можно найти API.
4. Доработать Atlantis – сейчас в `repos.yml` грубо перезаписывается процесс default, а можно организовать отдельный и подключать в `atlantis.yml`.
5. В Deployment забирать не фиксированный образ, а последний (а-то мы в `static-site` поправили, а раскатывается всё ещё старая версия).
6. При добавлении новых машин, k8s не раскатывается. Из-за этого те же группы виртуальных машин зависают, потому что новая машина не проходит healthcheck.
    * Да и в целом, если машина отвалится, то на новой не поднимется k8s, а зачем тогда всё это.
7. В `simple-vms` добавить default значения, а-то передавать много.
8. Тоньше настроить Atlantis – сейчас он ловит целую кучу событий, а реагирует не на все (push вообще не распознаёт).
