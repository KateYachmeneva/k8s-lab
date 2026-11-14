# Лабораторная работа № 1 — Kubernetes
## «Развёртывание приложения в Kubernetes»

Практическая работа: развернуть приложение в локальном Kubernetes-кластере,
настроить Deployment, Service и масштабирование, проверить самовосстановление.

## Как работать

1. Склонируйте репозиторий:
   ```bash
   git clone https://github.com/vladoky/k8s-lab.git
   cd k8s-lab
   ```
2. Выполните задания по порядку из папки `tasks/`. Начните с задания 0 —
   установка окружения.
3. Манифесты заполняйте в папке `templates/` (места `<ВПИШИТЕ ...>`
   с подсказками в комментариях).
4. Отчёт — по бланку `templates/lab-report.md`.

## Задания

| # | Тема | Файл |
|---|---|---|
| 0 | Установка окружения | [tasks/00-setup.md](tasks/00-setup.md) |
| 1 | Кластер: поднять и исследовать | [tasks/01-cluster.md](tasks/01-cluster.md) |
| 2 | Pod: запустить приложение | [tasks/02-pod.md](tasks/02-pod.md) |
| 3 | Deployment: управляемое приложение | [tasks/03-deployment.md](tasks/03-deployment.md) |
| 4 | Service: стабильный доступ | [tasks/04-service.md](tasks/04-service.md) |
| 5 | Scaling: изменить число реплик | [tasks/05-scaling.md](tasks/05-scaling.md) |
| 6 | Self-healing: самовосстановление | [tasks/06-self-healing.md](tasks/06-self-healing.md) |
| 7 | Разбор и закрепление | [tasks/07-review.md](tasks/07-review.md) |

## Структура репозитория

```
tasks/        инструкции к заданиям (выполнять по порядку)
templates/    шаблоны манифестов + бланк отчёта lab-report.md
app/          демо-приложение (калькулятор на Flask + Dockerfile)
```

## Что сдаётся

Заполненные манифесты + отчёт `lab-report.md` с выводами команд
и ответами на вопросы. Детали — в `templates/README.md`.
