# secrets/

Файлы в этом каталоге монтируются в контейнеры через Docker Compose `secrets:` как
`/run/secrets/<name>` (только для чтения). Приложение получает значение, читая файл
по пути из переменной окружения `*_FILE` — это стандартный 12-factor-ish паттерн
(`API_TOKEN_FILE=/run/secrets/api_token`), он же используется в официальных образах
postgres, mysql, redis и т.д.

## Важно

- **Файлы в этом каталоге — заглушки для демонстрации.** В реальном деплое их нужно
  заменить значениями, получаемыми из менеджера секретов (Docker Swarm secrets,
  HashiCorp Vault, AWS Secrets Manager, k8s Secrets и т.п.).
- **Не коммитьте реальные секреты в Git.** Для production создайте файлы с реальными
  значениями на целевом хосте и укажите их путь в `docker-compose.yml` (либо переведите
  на `external: true` + Swarm).
- В `.dockerignore` путь `secrets/` исключён из контекста сборки — секреты **никогда**
  не попадают в образ.

## Как заменить в проде

Вариант 1 — отдельные файлы на хосте:

```yaml
secrets:
  api_token:
    file: /etc/momo/api_token
```

Вариант 2 — Swarm / внешние секреты:

```yaml
secrets:
  api_token:
    external: true
```

После чего: `docker secret create api_token -`.
