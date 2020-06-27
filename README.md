# usecases

# one-liners
Для примеров ниже потрубуется утилита yc - https://cloud.yandex.ru/docs/cli/ 
В примерах ниже можно не использовать cloud_id, если при настройке yc был указан дефолтный cloud_id. Также этот идентификатор может быть найден на карточке облака в веб консоле https://console.cloud.yandex.ru/cloud или в выводе команды  `yc resource-manager cloud list`

1) полистить все инстансы во всех фолдерах
```bash
for id in $(yc resource-manager folder list --cloud-id $CLOUD_ID --format=json | jq -r .[].id); do yc compute instance list --folder-id=$id; done
```
2) полистить все фолдеры, в них все таргет группы балансировщика, чтобы найти в каком из них добавлен какой-нибудь адрес, который нужно найти
```bash
for id in $(yc resource-manager folder list --cloud-id $CLOUD_ID --format=json | jq -r .[].id); do for tgId in $(yc load-balancer target-group list --folder-id=$id --format=json | jq -r .[].id); do yc load-balancer target-group get --id $tgId; done; done | grep 10.61.240.23 -C 10
```
3) найти все инстансы с определенным кол-вом ядер
```
for i in $(yc resource-manager folder list --cloud-id=$CLOUD_ID --format=json | jq -r .[].id); do yc compute instance list --folder-id=$i --format json | jq ".[] | select(.resources.cores==\"8\")"; done
```
