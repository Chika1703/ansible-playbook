# Роль Vector

## Описание

Эта роль устанавливает и настраивает систему Vector для сбора и обработки логов.

## Переменные

- `vector_version`: версия Vector для установки (по умолчанию: `0.46.1`)
- `vector_arch`: архитектура системы (по умолчанию: `amd64`)
- `vector_config_path`: путь к конфигурационному файлу Vector (по умолчанию: `/etc/vector/vector.toml`)

## Пример использования

```yaml
- hosts: all
  roles:
    - vector-role
