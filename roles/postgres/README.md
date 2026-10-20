# Роль `postgres`

Устанавливает PostgreSQL 14 и PostgreSQL Exporter, создаёт пользователей и
базы данных, настраивает отдельный systemd-сервис и проверяет подключение
запросом `SELECT 1`.

## Требования

- Debian/Ubuntu с `systemd`;
- права `become`;
- коллекция `community.postgresql`.

## Переменные

- `postgres__packages` — устанавливаемые пакеты;
- `postgres__data_dir` — каталог данных PostgreSQL;
- `postgres__hba_rules` — правила `pg_hba.conf`;
- `postgres__opts` — параметры `postgresql.conf`;
- `postgres__users` — пользователи и их пароли;
- `postgres__databases` — базы данных и их владельцы;
- `postgres__healthcheck_database`, `postgres__healthcheck_user`,
  `postgres__healthcheck_password` — данные для проверки `SELECT 1`;
- `postgres__exporter_version`, `postgres__exporter_arch`,
  `postgres__exporter_port`, `postgres__exporter_user` — настройки экспортера.

Значения по умолчанию находятся в `defaults/main.yml`. Пароли следует хранить
в Ansible Vault.

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

Запуск:

```bash
ansible-playbook postgres.play.yml --ask-vault-pass
```
