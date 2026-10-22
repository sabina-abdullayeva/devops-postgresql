# Роль postgres

Роль поднимает свой экземпляр PostgreSQL с отдельным каталогом данных и
устанавливает postgres-exporter, стандартный кластер Ubuntu при этом
останавливается.

## Что можно настроить

- `postgres__packages` — пакеты для установки;
- `postgres__data_dir` — каталог с данными;
- `postgres__hba_rules` — содержимое `pg_hba.conf`;
- `postgres__opts` — параметры из `postgresql.conf`;
- `postgres__users` — пользователи PostgreSQL;
- `postgres__databases` — базы и их владельцы;
- `postgres__healthcheck_database` — база для `SELECT 1`;
- `postgres__healthcheck_user` — пользователь для проверки;
- `postgres__healthcheck_password` — его пароль;
- `postgres__exporter_version` — версия postgres-exporter;
- `postgres__exporter_arch` — архитектура архива;
- `postgres__exporter_port` — порт с метриками;
- `postgres__exporter_user` — пользователь экспортера.

Обычные настройки лежат в `defaults/main.yml`. В `vars/main.yml` находятся
внутренние значения роли с префиксом `__`.
Пароли в открытом виде хранить не надо, для них используется Ansible Vault.

## Пример

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

```bash
ansible-playbook postgres.play.yml --ask-vault-pass
```
