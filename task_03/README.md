# Лабораторная работа #03

<p align="center">Министерство образования Республики Беларусь</p>
<p align="center">Учреждение образования</p>
<p align="center">"Брестский государственный технический университет"</p>
<p align="center">Кафедра ИИТ</p>
<br><br><br><br><br>
<p align="center"><strong>Лабораторная работа #03</strong></p>
<p align="center"><strong>По дисциплине:</strong> "Распределенные системы и облачные технологии"</p>
<p align="center"><strong>Тема:</strong> Kubernetes: состояние и хранение</p>
<br><br><br><br><br>
<p align="right"><strong>Выполнил:</strong></p>
<p align="right">Студент 4 курса</p>
<p align="right">Группы АС-576</p>
<p align="right">Александров В. С.</p>
<p align="right"><strong>Проверил:</strong></p>
<p align="right">Несюк А. Н.</p>
<br><br><br><br><br>
<p align="center"><strong>Брест 2026</strong></p>

## Цель работы
Научиться работать со StatefulSet для управления stateful-приложениями, настроить постоянное хранилище через PVC/PV и StorageClass, создать Headless Service для стабильных DNS-имен, а также реализовать механизм резервного копирования и восстановления данных с помощью Job/CronJob.

### Персональный вариант
* **db:** postgres
* **pvc:** 5Gi
* **storageClass:** default
* **schedule:** "*/15 * * * *"
* **namespace:** state-web01
* **префиксы ресурсов:** db-, backup-, storage-

## Метаданные студента
* **ФИО:** Александров Владимир Сергеевич
* **Группа:** АС-576
* **№ студенческого (StudentID):** 2157425
* **Email (учебный):** zas57626@g.bstu.by
* **GitHub username:** RustyX-Kozlina
* **Вариант №:** w1
* **Дата выполнения:** 11.06.2026
* **ОС и версия:** Windows 10, Minikube v1.x, kubectl v1.x

## Структура проекта
```text
├── manifests/
│   ├── app-infra.yaml      # Инфраструктура (Namespace, Secret, PVC, StatefulSet, Headless Service, CronJob)
│   └── restore-job.yaml    # Одноразовый Job для восстановления БД из бэкапа
└── README.md               # Документация и отчёт ЛР03
```

## Подробное описание выполнения

### 1. Подготовка инфраструктуры хранения и СУБД
Создан манифест `app-infra.yaml`, включающий изолированное пространство имен `state-web01`, секреты `db-postgres-secret` для учетных записей PostgreSQL и постоянный том общего хранилища `backup-storage-pvc` емкостью 2Gi для хранения бэкапов. Развернут `StatefulSet` СУБД PostgreSQL версии 16 на базе Alpine с динамическим провижинингом тома данных `db-data-volume` на 5Gi (`storageClass: default`).

### 2. Сетевая идентификация и резервное копирование
Описан `Headless Service` (`clusterIP: None`) с именем `db-postgres-headless` для заданной DNS-адресации пода СУБД. Настроен `CronJob` резервного копирования, который запускается каждые 15 минут согласно варианту (`*/15 * * * *`), выполняет команду `pg_dump` и сохраняет слепок базы данных в изолированный персистентный том.

### 3. Восстановление данных
Создан отдельный манифест `restore-job.yaml`, запускающий задачу `Job`. Задача сбрасывает текущую схему базы данных и полностью восстанавливает таблицы из последнего актуального `.dump` файла, хранящегося в бэкап-томе.

## Запуск в Minikube
```bash
minikube.exe start --driver=docker
kubectl apply -f manifests/app-infra.yaml
```

## Проверка статусов и демонстрация работы

### 1. Вывод команды `kubectl get all -n state-web01`:
```text
NAME                                     READY   STATUS    RESTARTS   AGE
pod/db-postgres-0                        1/1     Running   0          7m19s

NAME                           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
service/db-postgres-headless   ClusterIP   None         <none>        5432/TCP   7m19s

NAME                           READY   AGE
statefulset.apps/db-postgres   1/1     7m19s

NAME                                      SCHEDULE       SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/backup-postgres-cronjob   */15 * * * *   False     0        <none>          7m19s
```

### 2. Проверка сохранности данных
Создаем тестовую таблицу и наполняем данными внутри пода СУБД:
```bash
kubectl exec -it db-postgres-0 -n state-web01 -- psql -U postgres -d stu_db_w1 -c "CREATE TABLE bstu_table (id SERIAL, val TEXT); INSERT INTO bstu_table (val) VALUES ('Alexandrov V.S.');"
# Вывод терминала:
# CREATE TABLE
# INSERT 0 1
```

Имитируем сбой/перезапуск пода:
```bash
kubectl delete pod db-postgres-0 -n state-web01
```

После автоматического пересоздания подов проверяем сохранность данных:
```bash
kubectl exec -it db-postgres-0 -n state-web01 -- psql -U postgres -d stu_db_w1 -c "SELECT * FROM bstu_table;"
# Вывод терминала:
#  id |       val       
# ----+-----------------
#   1 | Alexandrov V.S.
# (1 row)
```

### 3. Тестирование бэкапа и восстановления (Имитация аварии)
Принудительно вызываем джобу бэкапа вне расписания:
```bash
kubectl create job --from=cronjob/backup-postgres-cronjob backup-manual-run -n state-web01
```
Уничтожаем рабочую таблицу:
```bash
kubectl exec -it db-postgres-0 -n state-web01 -- psql -U postgres -d stu_db_w1 -c "DROP TABLE bstu_table;"
```
Применяем сценарий восстановления:
```bash
kubectl apply -f manifests/restore-job.yaml
```
Проверяем логи успешного восстановления:
```bash
kubectl logs job/backup-postgres-restore-job -n state-web01
# Вывод терминала:
# Starting restoration process...
# Data restoration successfully complete!
```

## Контрольный список (checklist)
- [x] 🟢 **Метаданные и аннотации вуза** — *(заданы в манифесте для StatefulSet и отчете)*
- [x] 🟢 **Выделенное пространство имен** — *(настроено ns: state-web01)*
- [x] 🟢 **Использование префиксов по ТЗ** — *(использованы префиксы db-, backup-, storage-)*
- [x] 🟢 **Конфигурация через StatefulSet** — *(СУБД запущена как стабильный stateful-компонент)*
- [x] 🟢 **Динамический PVC на 5Gi** — *(том успешно подключен через storageClass default)*
- [x] 🟢 **Headless Service** — *(clusterIP: None успешно настроен для сетевых имен)*
- [x] 🟢 **Бэкап через CronJob по расписанию** — *(настроена автоматизация pg_dump раз в 15 минут)*
- [x] 🟢 **Успешный тест восстановления данных** — *(отлажен запуск Job восстановления с проверкой логов)*

---

## Вывод
В ходе лабораторной работы было развернуто stateful-приложение (PostgreSQL) в Kubernetes с привязкой к постоянному хранилищу через `volumeClaimTemplates` (`5Gi`). Настроен стабильный DNS-доступ внутри кластера при помощи Headless-сервиса. Реализована стратегия аварийного восстановления: автоматизирован регулярный бэкап через `CronJob` и проверена процедура наката резервной копии через `Job` после симулированного удаления данных.
