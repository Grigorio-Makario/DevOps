В данном примере рассматривается установка и запуск K8s Cluster,
состоящего из:
1 - Master
1 - Worker
1 - Ingress
а также External Load Balancer. 
При необходимости расширения состава кластера (кол-во нод, подсетей и зон), необходимо внести соответствующие изенения в файле
/kubernetes_setup/terraform/k8s-cluster.tf
![image](https://github.com/Grigorio-Makario/DevOps/assets/119935857/0d9675aa-4888-42d0-b11a-3aefb05b900c)


----------------------------------------------------------
# Для работы с платформой необходимо создать свое “облако” 
на официальном сайте 

https://cloud.yandex.ru

## Установить Terraform 
Позволяет автоматизировать создание, обновление 
и удаление необходимых ресурсов.
Для использования необходимо установить клиент: 

https://learn.hashicorp.com/terraform/getting-started/install

## Для управление конфигурацией и развертывание приложений, необходимо установить Ansible

https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

## Установить kubectl – для управления k8s-кластером:

https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Установить Helm – для установки Helm-пакетов:

https://helm.sh/docs/intro/install/

## jq - консольная утилита для работы с JSON-данными:

https://stedolan.github.io/jq/

## Отдельно клонируем репозиторий Kubespray:
$ cd kubernetes_setup
$ git clone git@git.cloud-team.ru:ansible-roles/kubespray.git \
 kubespray -b cloudteam
Устанавливаем зависимости Kubespray:
$ sudo pip3 install -r kubespray/requirements.txt

## Определить необходимые переменные для Terraform
```
$ cp terraform/private.auto.tfvars.example terraform/private.auto.tfvars
$ vim terraform/private.auto.tfvars
```
##Необходимо заполнить следующие значения:
```
yc_token – OAuth-токен для доступа к API 
(https://cloud.yandex.ru/docs/iam/concepts/authorization/oauth-token)
yc_cloud_id – ID облака (скопировать из консоли управления)
yc_folder_id – ID каталога (скопировать из консоли управления)
```

## Для запуска установки и настройки кластера необходимо запустить следующий скрипт.
Скрипт выполняет следующие действия:
1. Запускает Terraform для создания необходимой инфраструктуры (она описана в файле “terraform/k8s-cluster.tf”).
2. Получает IP-адреса созданных виртуальных машин и генерирует файл hosts.ini для ansible-инвентаря.
3. Запускает Kubespray playbook со сгенерированным инвентарем для установки кластера на подготовленные сервера.

```
$ bash cluster_install.sh
```

## Копируем конфигурационный файл в каталог по умолчанию:
```
$ mkdir -p ~/.kube && cp kubespray/inventory/mycluster/artifacts/admin.conf ~/.kube/config
```


## Пример запуска pod-ов для тестового app + postgresql

```
$ helm install --namespace devops --create-namespace my-postgresql bitnami/postgresql -f values.yaml
$ helm upgrade -i --namespace devops  app App-HelmChart/
```
