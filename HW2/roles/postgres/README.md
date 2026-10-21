# Роль PostgreSQL

Роль устанавливает PostgreSQL, создаёт отдельный каталог данных и
systemd-службу, настраивает базы и пользователей, выполняет `SELECT 1`,
а также устанавливает postgres-exporter.

## Переменные

- `postgres__packages` — устанавливаемые пакеты;
- `postgres__data_dir` — каталог данных PostgreSQL;
- `postgres__hba_rules` — правила `pg_hba.conf`;
- `postgres__opts` — параметры `postgresql.conf`;
- `postgres__users` — пользователи PostgreSQL;
- `postgres__databases` — базы данных и их владельцы;
- `postgres__healthcheck_database` — база для проверки `SELECT 1`;
- `postgres__healthcheck_user` — пользователь для проверки;
- `postgres__healthcheck_password` — пароль пользователя для проверки;
- `postgres__exporter_version` — версия postgres-exporter;
- `postgres__exporter_arch` — архитектура postgres-exporter;
- `postgres__exporter_port` — порт метрик;
- `postgres__exporter_user` — пользователь postgres-exporter.

Значения по умолчанию находятся в `defaults/main.yml`. Внутренние
переменные роли находятся в `vars/main.yml` и имеют префикс `__`.
Пароли должны храниться в Ansible Vault.

## Пример playbook

```yaml
---
- name: Deploy PostgreSQL
  hosts: vm1
  become: true

  vars:
    postgres__users:
      - name: obito
        password: "{{ vault_obito_password }}"
    postgres__databases:
      - name: jinchuriki
        owner: obito
    postgres__healthcheck_database: jinchuriki
    postgres__healthcheck_user: obito
    postgres__healthcheck_password: "{{ vault_obito_password }}"

  roles:
    - postgres
```

## Запуск

```bash
ansible-playbook postgres.play.yml --ask-vault-pass
```
