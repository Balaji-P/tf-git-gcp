# Входные данные и требования

- У нас есть **один** GCP проект в рамках которого необходимо реализовать **отдельную** инфраструктуру для `production` и `qa` (`staging` и `review`) окружений.
- Необходимо разделить жизненые циклы инфраструктуры от кода самого сервиса.
- В рамках сервиса мы оперируем следующими окружениями:
  - `production` – название говорит само за себя.
  - `staging` – окружение в котором проверяем релиз перед тем как разварачивать его в `production` окружении.
  - `review` – динамическое окружение в котором разработчик может посмотреть результат своей работы не дожидаясь мерджа задачи в `master`.
- В рамках инфраструктуры мы оперируем следующими окружениями:
  - `production` – название говорит само за себя, используется для `production` окружения сервиса.
  - `qa` - окружение на котором предварительно проверятся изменения до применения их на `production`.  Используется для `staging` и `review` окружений сервиса.
- Оба `production` и `qa` окружения используют отдельный GKE кластер и Cloud SQL, но с разными параметрами. Например количеством нод в кластере и наличием реплики у Cloud SQL.

# Список окружений

Для управления инфраструктурой необходимо три окружения. Важно понимать что это не теже самые, которыми разработчики оперируют для своего сервиса:

- `production` - Long Lived. Название говорит само за себя. Используется для `production` окружения сервиса. Необходим GKE кластер и Cloud SQL с репликой.
- `staging` - Long Lived (необходимо понимать что произойдет с `production` окружением) Может быть поломан/недоступен. Используется для `staging` (используется реальный Cloud SQL) и `review` (используем Docker контейнер для MySQL) окружения сервиса. После `terraform apply` необходимо каким-либо образом перевылить сервис в том же состоянии в котором он последний раз выливался на `staging` окружение и ветки для которых было создано `review` окружение. В рамках кластера в этом окружении сервисы деплоятся в отдельные [Namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/).
- `review` - Short Lived. Необходимо что бы при желании можно было посмотреть на результат до мержа в ветку `staging`.

Поскольку у нас один GCP проект, каждое из окружений выносим в отдельный [VPC](https://cloud.google.com/vpc/).

# Git Environment Branches Flow

Для `production` и `staging` окружения используются [Environment Branches](https://docs.gitlab.com/ce/workflow/gitlab_flow.html#environment-branches-with-gitlab-flow). Порядок работы c репозиторием примерно следующий:

- При необходимости внести какие-либо изменения сначала откалываем новую ветку от `staging`.
- Делаем задачу.
- Открываем MR в ветку `staging`.
- Когда ветка задачи попадет в `staging` будет сгенерирован tfplan и после по кнопке можно его применить.
- Если со `staging` все ок - открываем MR на мерж ветки `staging` в `master`.

```
    master  : ----*-----------* (production)
                 /           /
                /           /
    staging : -*–––--------* (staging)
                \         /
                  \      /
    feature :       *---* (review/feature)
```

Таким образом напрямую в `master` пушить какие-либо изменения **запрещено**. 

В случае если необходимо внести какие-либо изменения для конкретного окружения - необходимо делать `git merge` руками используя стратегию [ours](https://git-scm.com/docs/merge-strategies#merge-strategies-ours). 

# Terraform

В зависимости от ветки в которую спушили изменения используем отдельные [Workspace](https://www.terraform.io/docs/state/workspaces.html) и соответсвенно отдельные tfstat. В качестве бекенда используем [GCS](https://www.terraform.io/docs/backends/types/gcs.html) (есть возможность [лочить](https://www.terraform.io/docs/state/locking.html)).

- `project` – глобальные настройки проекта вроде IAM политик. Применятся только при пуше в `master`.
- `production` – аналогично при пуше в `master`.
- `staging` – при пуше в `staging`.
- review-xxx – создается динамически при необходимости развернуть review окружение для задачи. После удаляется.

## Структура проекта

Пока цели вынести [модули](https://www.terraform.io/docs/configuration/modules.html) в отдельный проект нет, но всесте с тем это позволит структурировать код.  Храним все в одной репе. 

```
global/
  project/     # Global GCP project settings (global workspace)
    main.tf
modules/
  gcp/
    services/  # GCP enabled API services
    iam/       # IAM policy
    vpc/       # VPC Network
    gke/       # Kubernetes
    sql/       # Cloud SQL
services/
  comments/
    main.tf    # Environment    (production & qa workspace)
      module "gke" ...
      module "sql" ... 
```