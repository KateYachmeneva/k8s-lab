# Инструкция. Задание 6 — Self-healing: самовосстановление

**Цель:** убедиться, что контроллеры Kubernetes сами возвращают кластер
к желаемому состоянию — на двух уровнях: под и контейнер.

**Понадобится:** задания 3–5 выполнены (deployment 3/3).

---

## Ход работы

### Шаг 1. Подготовка наблюдения

Во втором терминале:
```bash
kubectl get pods -n demo -w
```

### Шаг 2. Удаляем под

В первом терминале:
```bash
kubectl get pods -n demo                    # выберите имя любого пода
kubectl delete pod <имя-пода> -n demo
```

Во втором терминале — сразу видно:
```
calc-app-...-aaaaa   Terminating
calc-app-...-ddddd   Pending
calc-app-...-ddddd   ContainerCreating
calc-app-...-ddddd   Running
```

```bash
kubectl get pods -n demo
```
Ожидается: снова 3 пода Running, но у нового пода **другое имя и другой IP**.
Сравните с IP из задания 4.

✔ **Контрольная точка:** под пересоздан ReplicaSet'ом автоматически.

### Шаг 3. «Падает» процесс внутри контейнера

```bash
POD=$(kubectl get pod -n demo -l app=calc-app -o name | head -1)
kubectl exec -n demo $POD -- kill 1
```
`exit code 137` — это нормально: мы отправили SIGKILL главному процессу.

```bash
kubectl get pods -n demo
```
Ожидается: под **тот же** (имя и IP не изменились), но в колонке
**RESTARTS** у него прибавилась единица — kubelet перезапустил контейнер
(restartPolicy: Always).

✔ **Контрольная точка:** вы видите разницу двух механизмов:
пересоздание пода (шаг 2) vs перезапуск контейнера (шаг 3).

### Шаг 4. Хронология событий

```bash
kubectl get events -n demo --sort-by=.lastTimestamp | tail -10
```
Найдите события ваших экспериментов: `Killing`, `Scheduled`, `Created`,
`Started`, `Pulled`.

### Шаг 5. Граница самовосстановления

```bash
kubectl delete deployment calc-app -n demo
kubectl get pods -n demo
```
Ожидается: поды удаляются и **не восстанавливаются** — желаемое
состояние теперь «нет deployment». Kubernetes восстанавливает не
«всё подряд», а именно desired state.

Верните приложение:
```bash
kubectl apply -f templates/deployment.yaml
kubectl rollout status deployment/calc-app -n demo
```

---

## Что сдать в отчёте

1. Скриншот терминала с `-w` в момент пересоздания пода
2. Вывод `kubectl get pods` после `kill 1` с выросшим RESTARTS
3. Ответы: кто восстанавливает под, а кто — контейнер?
   Почему после удаления deployment ничего не восстановилось?

## Частые проблемы

| Симптом | Решение |
|---|---|
| RESTARTS не растёт | убивали не тот под / смотрели не тот; проверьте имя |
| `error: cannot exec` | под ещё не Running — подождите |
| Под не пересоздался | удалили «голый» под из задания 2, а не под deployment'а |
