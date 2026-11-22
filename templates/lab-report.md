# Отчёт по лабораторной работе № 1
## «Развёртывание приложения в Kubernetes»

**Студент:** Ячменева Екатерина Викторовна
**Группа:** ____________________
**Дата:** 15 ноября 2026

---

## Задание 1. Кластер

Кластер уже был поднят (`minikube`, драйвер docker). Повторный старт не требовался: узел в статусе `Ready`.

`kubectl cluster-info`:

```
Kubernetes control plane is running at https://127.0.0.1:60587
CoreDNS is running at https://127.0.0.1:60587/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

- Вывод `kubectl get nodes`:

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   29h   v1.37.0
```

- Вывод `kubectl get nodes -o wide`:

```
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION             CONTAINER-RUNTIME
minikube   Ready    control-plane   29h   v1.37.0   192.168.49.2   <none>        Debian GNU/Linux 12 (bookworm)   6.10.14-linuxkit (arm64)   containerd://2.3.4
```

Исследование узла (`kubectl describe node minikube`):

- Версия Kubernetes: **v1.37.0** (kubelet). Container runtime: **containerd 2.3.4**.
- Capacity: **10 CPU**, память **11205692Ki** (около 10,7 ГиБ), до 110 подов.
- Conditions (все проверки пройдены):
  - `MemoryPressure=False` — памяти достаточно
  - `DiskPressure=False` — диск не переполнен
  - `PIDPressure=False` — процессов не слишком много
  - `Ready=True` — kubelet сообщает, что узел готов

Системные поды (`kubectl get pods -A`), все `Running`:

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   coredns-559f6c778d-nx9bc           1/1     Running   0          29h
kube-system   etcd-minikube                      1/1     Running   0          29h
kube-system   kindnet-lpx22                      1/1     Running   0          29h
kube-system   kube-apiserver-minikube            1/1     Running   0          29h
kube-system   kube-controller-manager-minikube   1/1     Running   0          29h
kube-system   kube-proxy-b72np                   1/1     Running   0          29h
kube-system   kube-scheduler-minikube            1/1     Running   0          29h
kube-system   storage-provisioner                1/1     Running   0          29h
```

Четыре компонента control plane на месте: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`.

- Сколько нод в кластере и почему?

Одна нода — `minikube`. Локальный minikube по умолчанию поднимает одноузловой кластер: control plane и рабочие поды живут на одной машине. Для лабораторной этого достаточно, отдельные worker-ноды не создавались.

## Задание 2. Pod

- Команда, которой запущен под:

```
kubectl run calc-pod --image=calc-app:v1 --image-pull-policy=Never --port=5000
```

- Вывод `kubectl get pods -o wide` (под работал):

```
NAME       READY   STATUS    RESTARTS   AGE   IP           NODE
calc-pod   1/1     Running   0          2s    10.244.0.7   minikube
```

Events в `kubectl describe pod calc-pod`: `Scheduled` → `Pulled` → `Created` → `Started`.

Лог Flask: `Running on all addresses (0.0.0.0)`, также `http://10.244.0.3:5000`.

Внутри контейнера (`kubectl exec`, каталог `/srv`): `app.py`, `calculator.py`, `requirements.txt`.

- Ответ приложения на `/calc?op=mul&a=6&b=7`: `{"result":42.0}`
- `/health`: `{"status":"ok"}`
- Порт 5000 на компьютере был занят, поэтому port-forward шёл на `127.0.0.1:5001`.

Манифест `templates/pod.yaml` заполнен: имя `calc-pod`, label `app: calc-app`, образ `calc-app:v1`, `imagePullPolicy: Never`, порт `5000`. `kubectl apply --dry-run=client` прошёл без ошибок.

- Что произошло после `kubectl delete pod`? Почему?

Под исчез и не появился снова: `No resources found in default namespace`. За голым Pod никто не следит. `restartPolicy` перезапускает только упавший контейнер внутри существующего пода, а удалённый объект Pod некому создать заново. Это делает Deployment, не сам Pod.

## Задание 3. Deployment

- Вывод `kubectl get all -n demo`:

```
NAME                            READY   STATUS    RESTARTS   AGE
pod/calc-app-6f4d77fd67-92x85   1/1     Running   0          2m21s
pod/calc-app-6f4d77fd67-njrpc   1/1     Running   0          2m21s
pod/calc-app-6f4d77fd67-vspm6   1/1     Running   0          2m21s

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calc-app   3/3     3            3           2m21s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/calc-app-6f4d77fd67   3         3         3       2m21s
```

Rollout завершился: `deployment "calc-app" successfully rolled out`.

- Объясните имена подов: из каких частей они состоят?

Имя пода устроено как `<deployment>-<хэш ReplicaSet>-<хэш пода>`.

Пример: `calc-app-6f4d77fd67-92x85`

- `calc-app` — имя Deployment;
- `6f4d77fd67` — хэш ReplicaSet (pod-template-hash), он один и тот же у всех трёх подов и совпадает с именем `replicaset.apps/calc-app-6f4d77fd67`;
- `92x85`, `njrpc`, `vspm6` — случайный суффикс конкретного пода.

- Кто кем управляет?

Deployment `calc-app` управляет ReplicaSet `calc-app-6f4d77fd67`. ReplicaSet управляет подами: держит их число равным 3. Сами поды никого не создают. Поэтому цепочка такая: Deployment → ReplicaSet → Pod.

- Обновление до v2

Выполнены `docker tag calc-app:v1 calc-app:v2`, `minikube image load calc-app:v2`, `kubectl set image deployment/calc-app calc-app=calc-app:v2 -n demo`. Ответ: `deployment.apps/calc-app image updated`. `kubectl rollout status` закончился строкой `deployment "calc-app" successfully rolled out`. Образ сейчас `calc-app:v2`, поды пересозданы новым ReplicaSet `calc-app-58b78d4568`.

- Что показал `kubectl rollout history` до отката?

Две ревизии: `1` и `2`, у обеих `CHANGE-CAUSE` пустой (`<none>`). Ревизия 1 — образ `calc-app:v1`, ревизия 2 — `calc-app:v2`.

- Откат

`kubectl rollout undo deployment/calc-app -n demo` ответил `rolled back`. `kubectl rollout status` закончился строкой `deployment "calc-app" successfully rolled out`.

Текущее состояние: образ снова `calc-app:v1`, три пода у ReplicaSet `calc-app-6f4d77fd67`. ReplicaSet версии v2 (`calc-app-58b78d4568`) остался, но масштабирован в 0.

После отката `rollout history` показывает ревизии `2` и `3`, а не `1` и `2`. Откат не возвращает номер 1: Kubernetes записывает его как новую ревизию 3 с шаблоном v1. Ревизия 2 — это оставшийся `calc-app:v2`. `CHANGE-CAUSE` по-прежнему `<none>`.

- Зачем нужен промежуточный ReplicaSet между Deployment и Pod?

Deployment не создаёт поды сам. Он создаёт ReplicaSet на каждый шаблон пода. ReplicaSet держит нужное число подов именно этой версии. Когда образ меняется, появляется новый ReplicaSet, а старый остаётся с нулём подов. Поэтому откат возможен: Deployment снова поднимает прежний ReplicaSet, а не собирает поды заново из ничего.

## Задание 4. Service

`kubectl get services -n demo`:

```
NAME       TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
calc-app   NodePort   10.98.114.75   <none>        5000:30500/TCP   9s
```

`kubectl describe service calc-app -n demo`:

- Selector: `app=calc-app`
- Endpoints: `10.244.0.10:5000,10.244.0.11:5000,10.244.0.9:5000`

Это IP всех трёх подов. Селектор совпал с меткой подов.

- Ответ `curl` по DNS-имени из тестового пода:

`curl "http://calc-app.demo:5000/calc?op=add&a=2&b=3"` вернул `{"result":5.0}`.

- Чем вы объясняете, что DNS-имя `calc-app.demo` работает изнутри кластера?

Имя собирается как `<сервис>.<namespace>`. CoreDNS в кластере резолвит его в ClusterIP сервиса `10.98.114.75`, а не в IP конкретного пода. Поэтому имя не меняется, когда поды пересоздаются.

- Как Service узнаёт, какие поды живы?

По селектору `app=calc-app` он находит поды Deployment. В Endpoints попадают только готовые поды и их `IP:5000`. Если под умирает, его адрес из Endpoints пропадает, и трафик на него больше не идёт.

Балансировка: в `kubectl logs -n demo -l app=calc-app --prefix=true` запросы попали на три разных пода — `2ggb7`, `hg2gr` и `nljv5`.

## Задание 5. Scaling

- Что показал `kubectl describe service` после scale до 5 реплик?

`kubectl get deployment` показал `READY 5/5`. Endpoints:

```
10.244.0.10:5000,10.244.0.11:5000,10.244.0.13:5000 + 2 more...
```

Пять адресов. Service не меняли: новые поды получили ту же метку `app=calc-app`, и Endpoints обновился сам.

Дальше `kubectl scale --replicas=2`, затем `kubectl apply -f templates/deployment.yaml`. В файле `replicas: 3`, поэтому сразу после apply было `2/3` при `UP-TO-DATE 3`: желаемое число уже 3, третий под ещё поднимался.

- В чём разница между `kubectl scale` и правкой YAML + apply? Какой способ для production?

`kubectl scale` меняет живой объект в кластере и не трогает файл. `kubectl apply` читает YAML и приводит кластер к нему. Это видно по ходу работы: после `scale --replicas=2` следующий `apply` файла с `replicas: 3` вернул желаемое число к 3.

Для production правилен `apply`: желаемое состояние лежит в файле и в git. `scale` удобен для быстрой проверки, но следующий `apply` затрёт его значением из файла.

- Эксперимент с 50 репликами

Эксперимент повторён: `kubectl scale deployment calc-app --replicas=50 -n demo`. Сразу после команды 44 пода были `Pending`. Для одного из них, `calc-app-6f4d77fd67-28bcp`, выполнен `kubectl describe pod`. В Events нет `Insufficient cpu`. Планировщик записал: `Successfully assigned demo/calc-app-6f4d77fd67-28bcp to minikube`. Поды не запрашивают CPU, а нода вмещает до 110 подов, поэтому все 50 были приняты. Через несколько секунд Deployment стал `50/50`. Затем `kubectl scale --replicas=3` вернул его к `3/3`, как в `templates/deployment.yaml`.

## Задание 6. Self-healing

- Что наблюдалось после удаления пода

Во втором окне был `kubectl get pods -n demo -w`. В первом удалён `calc-app-6f4d77fd67-2ggb7` (`pod "calc-app-6f4d77fd67-2ggb7" deleted`).

В `-w` этот под ушёл в `Terminating`, а ReplicaSet создал новый `calc-app-6f4d77fd67-hrbtz`: `Pending` → `ContainerCreating` → `Running`. Следующий `kubectl get pods` снова показал три пода `Running`. Новому было 38 секунд, `74gxc` и `hg2gr` остались прежними. Адрес нового пода — `10.244.0.110`.

В событиях ReplicaSet `calc-app-6f4d77fd67`: `SuccessfulCreate`, `Created pod: calc-app-6f4d77fd67-hrbtz`, затем `Pulled` и `Started`.

- crash-demo и RESTARTS

```
kubectl run crash-demo --image=python:3.12-slim --restart=Always -n demo -- \
  python3 -c "import time; time.sleep(3); exit(1)"
```

Ответ: `pod/crash-demo created`. Процесс спит 3 секунды и завершается с кодом 1.

В `-w` имя пода не менялось, а колонка RESTARTS росла: 0 → 1 → 2 → 3 → 4. Статус шёл `Running` → `Error` → `CrashLoopBackOff`. `kubectl get pods` поймал строку `crash-demo 0/1 Error 2 (19s ago) 30s`. В событиях есть `Warning BackOff`: `Back-off restarting failed container`.

`CrashLoopBackOff` — нарастающая пауза между перезапусками, а не поломка кластера. После наблюдения под удалён: `pod "crash-demo" deleted`.

- Кто восстанавливает под, а кто — контейнер

Под восстанавливает ReplicaSet: после удаления появился новый под с другим именем. Контейнер перезапускает kubelet по `restartPolicy`: под `crash-demo` остался тем же, росла колонка RESTARTS.

- Граница самовосстановления

`kubectl delete deployment` в этом сеансе не выполнялся. `kubectl apply -f templates/deployment.yaml` запущен из каталога `~`, поэтому ответ: `the path "templates/deployment.yaml" does not exist`. Deployment остался прежним (возраст около 8 часов, `3/3`), и `kubectl rollout status` сразу ответил `deployment "calc-app" successfully rolled out`.

Kubernetes возвращает желаемое состояние. Пока в нём есть Deployment `calc-app` с тремя репликами, удалённый под создаётся снова. Если удалить сам Deployment, желаемым состоянием станет отсутствие приложения, и поды не вернутся, пока его не создадут заново через `apply`.

## Задание 7. Разбор

- Цепочка от YAML до ответа

`templates/deployment.yaml` задаёт желаемое: Deployment `calc-app` в namespace `demo`, `replicas: 3`, метка `app=calc-app`, образ `calc-app:v1`, порт 5000. `kubectl apply` отдаёт манифест API-серверу, тот записывает объект в etcd. Контроллер Deployment создаёт ReplicaSet — для v1 это `calc-app-6f4d77fd67`. Планировщик ставит поды на узел `minikube`, kubelet через containerd запускает контейнер, под становится `Running`. Service с тем же selector собирает Endpoints из готовых подов. Запрос на имя `calc-app.demo:5000` пришёл на ClusterIP `10.98.114.75` и вернул `{"result":5.0}`. Умножение `6×7` через port-forward вернуло `{"result":42.0}`.

- Чем Pod отличается от контейнера?

Pod — объект Kubernetes: его планировщик ставит на узел, у него свой IP и метки. Контейнер — процесс внутри пода. У `crash-demo` имя пода не менялось, а kubelet заново запускал контейнер: колонка RESTARTS выросла с 0 до 4. Когда удалили под `calc-app-6f4d77fd67-2ggb7`, ReplicaSet создал другой под, `calc-app-6f4d77fd67-hrbtz`, с адресом `10.244.0.110`.

- Зачем нужен ReplicaSet, если есть Deployment?

Deployment не создаёт поды сам. На каждый шаблон он заводит ReplicaSet, и уже ReplicaSet держит число копий. Старый ReplicaSet остаётся с нулём, поэтому откат поднимает его снова. Так и было: `calc-app-6f4d77fd67` держал образ `calc-app:v1`, `calc-app-58b78d4568` — `calc-app:v2` и после отката стоял на 0. Событие `SuccessfulCreate` при создании `hrbtz` пришло от ReplicaSet.

- Почему нельзя обращаться к подам напрямую по IP?

IP пода исчезает вместе с подом. Удалённого `2ggb7` больше нет, новый `hrbtz` получил `10.244.0.110`. У Service адрес постоянный: ClusterIP `10.98.114.75` и DNS-имя `calc-app.demo`. Endpoints сам подставляет IP готовых подов. Запрос по имени сервиса вернул `{"result":5.0}`.

- Что такое desired state и кто его поддерживает?

Desired state — запись в API о том, что должно работать: три реплики, образ `calc-app:v1`, сервис с меткой `app=calc-app`. Её поддерживают контроллеры. ReplicaSet вернул третий под после `delete`. kubelet перезапускал контейнер `crash-demo` по `restartPolicy: Always`. `kubectl apply` файла с `replicas: 3` вернул число реплик после `scale`.

- Что изолирует namespace, а что — нет?

Namespace `demo` отделяет имена объектов и DNS: сервис называется `calc-app.demo`, его поды не попадают в список `kube-system`. Узел общий. Все поды стояли на `minikube`, сеть подов одна (`10.244.0.0`), ёмкость узла тоже одна: 10 CPU и до 110 подов. Квоты и NetworkPolicy в работе не задавались.

- Под не стартует. Три команды диагностики

`kubectl describe pod` — события (`Scheduled`, `Pulled`, `Started`, `BackOff`). `kubectl logs` — вывод процесса, у калькулятора это строки Flask. `kubectl get events -n demo --sort-by=.lastTimestamp` — общая лента: `Killing`, `SuccessfulCreate`, `BackOff`.

- Чек-лист

Все 12 команд выполнены по ходу работы:

1. `kubectl get nodes` — узел `minikube` в статусе `Ready`, версия v1.37.0.
2. `kubectl run` с образом `calc-app:v1` — под `calc-pod`. После `delete` он не вернулся: у голого пода нет контроллера.
3. `kubectl apply -f templates/deployment.yaml` — Deployment `3/3`.
4. `kubectl get pods -n demo -w` — видно `Terminating` у `2ggb7` и появление `hrbtz`.
5. `kubectl describe pod` — события планировщика и kubelet.
6. `kubectl logs` — лог Flask и запросы на поды.
7. `kubectl exec` — внутри контейнера в `/srv` лежат `app.py`, `calculator.py`, `requirements.txt`.
8. `kubectl port-forward` — ответ `{"result":42.0}` на `127.0.0.1:5001`.
9. `kubectl apply -f templates/service.yaml` — Service `NodePort`, `10.98.114.75`, порты `5000:30500`.
10. `kubectl scale` — проверялись 5, 2, 50 и возврат к 3.
11. `kubectl delete pod` — ReplicaSet создал новый под.
12. `kubectl rollout history` перед удалением namespace показывал ревизии 2 и 3, у обеих `CHANGE-CAUSE` пустой (`<none>`).

Самооценка «без подглядывания» числом не ставится: команды выполнялись по тексту заданий. Факт выполнения — 12 из 12.

- Уборка

```
kubectl delete namespace demo
namespace "demo" deleted

kubectl get all -n demo
No resources found in demo namespace.
```

В списке namespace остались `default`, `kube-node-lease`, `kube-public`, `kube-system`. Узел `minikube` по-прежнему `Ready`.

## Вывод

Желаемое состояние — это запись в etcd о том, каким должен быть кластер: три реплики `calc-app` и сервис с меткой `app=calc-app`. Kubernetes называют декларативной системой, потому что в YAML описывается этот результат, а контроллеры сами приводят к нему факты: ReplicaSet вернул удалённый под, kubelet перезапускал упавший контейнер, `apply` вернул число реплик к значению из файла. Команда `kubectl delete namespace demo` убрала и саму запись, и все объекты, которые из неё следовали. Одной командой `kubectl get all -n demo` после этого получил ответ `No resources found`.

---

**Самооценка по чек-листу** (`tasks/07-review.md`): 12 из 12 команд выполнены по ходу работы. Отметка «без подглядывания» не проставлена: команды шли по тексту заданий.
