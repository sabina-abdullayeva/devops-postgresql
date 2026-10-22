# HW2 — PostgreSQL через Ansible
Роль:

- устанавливает PostgreSQL из репозиториев Ubuntu;
- создаёт кластер в `/data/postgres/data`;
- настраивает `postgresql.conf` и `pg_hba.conf`;
- создаёт пользователей и базы;
- проверяет подключение запросом `SELECT 1`;
- запускает PostgreSQL и postgres-exporter через systemd.

## Запуск

Нужны Ubuntu/Debian, доступ `become` и коллекция
`community.postgresql`.

```bash
ansible-playbook postgres.play.yml --ask-vault-pass
```

Пароль пользователя `obito` лежит в Ansible Vault:
`inventories/main/group_vars/all.yml`.

Переменные и пример использования роли описаны отдельно в
[`roles/postgres/README.md`](roles/postgres/README.md).

## Проверка

После запуска проверка службы, подключение к PostgreSQL, базы,
пользователей и метрики `pg_up`.

Логи:
 повторный запуск с `changed=0`.
