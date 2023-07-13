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
```
$ bash cluster_install.sh
```

## Copy generated config
```
$ mkdir -p ~/.kube && cp kubespray/inventory/mycluster/artifacts/admin.conf ~/.kube/config
```

## Deploy test app
```
$ kubectl apply -f manifests/test-app.yml
```

## Add hosts to your local hosts file
```
$ sudo sh -c "cat kubespray_inventory/etc-hosts >> /etc/hosts"
```

## Check external access to test app
```
$ curl hello.local
Hello from my-deployment-784598767c-7gjjs
```

# Cluster monitoring

## Install Kubernetes Dashboard
```
$ helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
$ helm install --namespace monitoring --create-namespace -f manifests/dashboard-values.yml \
  kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard
$ kubectl apply -f manifests/dashboard-admin.yml
$ kubectl -n monitoring describe secret \
  $(kubectl -n monitoring get secret | grep admin-user | awk '{print $1}')
$ kubectl port-forward -n monitoring $(kubectl get pods -n monitoring \
  -l "app.kubernetes.io/name=kubernetes-dashboard" -o jsonpath="{.items[0].metadata.name}") 9090
```
Go to http://localhost:9090 and use token for authentication

## Install Prometheus and Grafana
```
$ helm install --namespace monitoring --create-namespace -f manifests/prometheus-values.yml \
  prometheus stable/prometheus
$ helm install --namespace monitoring --create-namespace -f manifests/grafana-values.yml \
  grafana stable/grafana
```

### Access Prometheus UI

Go to http://prometheus.local

### Access Grafana UI
```
$ kubectl get secret -n monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Go to http://grafana.local (user: admin, password: result of first command).
Add new data source with type "Prometheus" and url "http://prometheus-server".
Import a new dashboard to Grafana (grafana.com dashboard: https://grafana.com/dashboards/1621, Prometheus: created one).

# Logging

## Deploy Loghouse
```
$ helm repo add loghouse https://flant.github.io/loghouse/charts/
$ helm install --namespace loghouse --create-namespace -f manifests/loghouse-values.yml \
  loghouse loghouse/loghouse
```
Go to http://loghouse.local (login: admin, password: PASSWORD).

Try to search logs of test app with the query:
```
~app = "my-app"
```

# Cluster backup/restore

## Install Velero

https://velero.io/docs/v1.4/basic-install/

## Install and configure AWS plugin
```
$ velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.1.0 \
  --bucket backup-backet \
  --backup-location-config region=ru-central1-a,s3ForcePathStyle="true",s3Url=https://storage.yandexcloud.net \
  --snapshot-location-config region=ru-central1-a \
  --secret-file kubespray_inventory/credentials-velero
```

## Create backup and watch its status
```
$ velero backup create my-first-backup
$ velero backup get
```

## Delete test app
```
$ kubectl delete -f manifests/test-app.yml
```

## Restore backup and list restores
```
$ velero restore create --from-backup my-first-backup
$ velero restore get
```

# Destroy cluster

## Delete cloud resources
```
$ bash cluster_destroy.sh
```
