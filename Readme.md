**Домашнее задание «Работа с roles»**

## **Студент:** *Колб Дмитрий*

## Содержание

- [**Студент:** *Колб Дмитрий*](#студент-колб-дмитрий)
- [Содержание](#содержание)
- [Введение](#введение)
- [Задания](#задания)
- [Описание проекта](#описание-проекта)
- [Структура репозиториев и важные файлы](#структура-репозиториев-и-важные-файлы)
  - [vector-role](#vector-role)
  - [playbook (основной репозиторий)](#playbook-основной-репозиторий)
- [Выполнение](#выполнение)
- [Скриншоты](#скриншоты)
- [Заключение](#заключение)

---

## Введение

В рамках этого домашнего задания требовалось разбить готовый playbook на отдельные `roles` для трёх компонентов: ClickHouse, Vector и Lighthouse.

1. Роль для ClickHouse была взята из внешнего репозитория (AlexeySetevoi/ansible-clickhouse).
2. Для Vector реализована самостоятельная роль `vector-role`.
3. Для Lighthouse вместо самостоятельной роли использована готовая роль из Ansible Galaxy `consensys.lighthouse`, и добавлены некоторые элементы.

После создания необходимых ролей собран единый playbook, который развёртывает эти компоненты на трёх группах хостов.

---

## Задания

1. Создать в исходном playbook файл `requirements.yml` и скачать роль ClickHouse:

   ```yaml
   ---
   - src: git@github.com:AlexeySetevoi/ansible-clickhouse.git
     scm: git
     version: "1.13"
     name: clickhouse
   ```
2. С помощью `ansible-galaxy install -r requirements.yml` установить роль ClickHouse.
3. Создать каталог и роль для Vector:

   ```bash
   ansible-galaxy role init vector-role
   ```

   — перенести в неё задачи и шаблоны из старого playbook, разделить переменные между `defaults/` и `vars/`, положить шаблоны в `templates/`.
4. Описать в `README.md` самодостаточную документацию для роли `vector-role`.
5. Повторить шаги 3–6 для роли Lighthouse. Для Lighthouse была выбрана существующая роль `consensys.lighthouse` из Galaxy (см. [https://galaxy.ansible.com/consensys/lighthouse](https://galaxy.ansible.com/consensys/lighthouse)).
6. Опубликовать все роли в отдельные репозитории (у меня:

   * `vector-role` — ­[https://github.com/Chika1703/vector-role](https://github.com/Chika1703/vector-role)
   * Lighthouse (используется внешняя роль Ansible Galaxy, репозиторий не создавался)
   * ClickHouse (взята из AlexeySetevoi/ansible-clickhouse)
7. Прописать в основном репозитории playbook файл `requirements.yml` с перечислением трёх ролей:

   ```yaml
   ---
   - src: git@github.com:Chika1703/vector-role.git
     scm: git
     version: "1.0.0"
     name: vector

    - src: consensys.lighthouse
      version: 0.0.13

   - src: git@github.com:AlexeySetevoi/ansible-clickhouse.git
     scm: git
     version: "1.13"
     name: clickhouse
   ```
8. Переработать главный playbook с учётом новых ролей и зависимостей.
9. Опубликовать готовый playbook в отдельный репозиторий.

---

## Описание проекта

В итоге получился единый playbook, который:

1. Устанавливает ClickHouse на группу `clickhouse`.
2. Устанавливает и настраивает Vector на группу `vector`.
3. Устанавливает Lighthouse (с помощью роли `consensys.lighthouse`) на группу `lighthouse`.

Для каждой группы предусмотрены свои переменные, шаблоны и обработчики. Playbook выполнен идемпотентно и прошёл проверку через `ansible-lint`.

---

## Структура репозиториев и важные файлы

### vector-role

**Репозиторий:** [https://github.com/Chika1703/vector-role](https://github.com/Chika1703/vector-role)
**Структура:**

```
vector-role/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   └── vector-config.toml.j2
├── vars/
│   └── main.yml
└── README.md
```

* **defaults/main.yml**

  ```yaml
  ---
  vector_version: "0.46.1"
  vector_arch: "amd64"
  vector_config_dir: "/etc/vector"
  vector_config_file: "{{ vector_config_dir }}/vector.toml"
  ```
* **vars/main.yml**

  ```yaml
  ---
  vector_service_name: "vector"
  ```
* **tasks/main.yml**

  ```yaml
  ---
  - name: Ensure apt-transport-https is installed
    apt:
      name: apt-transport-https
      state: present
    become: true

  - name: Download Vector .deb package
    get_url:
      url: "https://apt.vector.dev/pool/v/ve/vector_{{ vector_version }}-1_{{ vector_arch }}.deb"
      dest: "/tmp/vector_{{ vector_version }}.deb"
      mode: '0644'
    become: true

  - name: Install Vector package
    apt:
      deb: "/tmp/vector_{{ vector_version }}.deb"
      state: present
    become: true

  - name: Create Vector configuration directory
    file:
      path: "{{ vector_config_dir }}"
      state: directory
      mode: '0755'
    become: true

  - name: Deploy Vector configuration
    template:
      src: "vector-config.toml.j2"
      dest: "{{ vector_config_file }}"
      mode: '0644'
    notify: Restart Vector service
    become: true

  - name: Ensure Vector service is enabled and started
    service:
      name: "{{ vector_service_name }}"
      enabled: true
      state: started
    become: true
  ```
* **handlers/main.yml**

  ```yaml
  ---
  - name: Restart Vector service
    service:
      name: "{{ vector_service_name }}"
      state: restarted
  ```
* **templates/vector-config.toml.j2**

  ```toml
  [sources.stdin]
  type = "stdin"

  [sinks.console]
  inputs = ["stdin"]
  type = "console"
  encoding.codec = "json"
  ```
* **README.md**
  Описание роли, переменных и пример использования:

  ````markdown
  # Роль Vector

  ## Описание
  Данная роль устанавливает и настраивает Vector (версия по умолчанию: 0.46.1) для сбора и обработки логов.

  ## Переменные (defaults)
  - `vector_version`: версия Vector (по умолчанию: "0.46.1")
  - `vector_arch`: архитектура пакета (по умолчанию: "amd64")
  - `vector_config_dir`: директория для конфигурации Vector (по умолчанию: "/etc/vector")
  - `vector_config_file`: полный путь до файла конфигурации Vector

  ## Переменные (vars)
  - `vector_service_name`: имя службы Vector (по умолчанию: "vector")

  ## Пример использования
  ```yaml
  - hosts: vector
    become: true
    roles:
      - role: vector-role
        vector_version: "0.46.1"
        vector_arch: "amd64"
  ````

  ```
  ```

---

### playbook (основной репозиторий)

**Репозиторий:** [https://github.com/Chika1703/ansible-playbook](https://github.com/Chika1703/ansible-playbook)
**Структура:**

```
ansible-playbook/
├── inventory/
│   └── prod.yml
├── roles/
│   ├── clickhouse            # установленная внешняя роль (AlexeySetevoi/ansible-clickhouse)
│   ├── vector                # ваша роль vector-role
│   └── consensys.lighthouse  # роль из Ansible Galaxy
├── site.yml
├── requirements.yml
└── README.md
```

* **inventory/prod.yml**

  ```yaml
  lighthouse:
    hosts:
      deb-1:
        ansible_host: 81.19.136.70

  vector:
    hosts:
      el-1:
        ansible_host: 81.19.136.48

  clickhouse:
    hosts:
      deb-1:
        ansible_host: 81.19.136.70
  ```

  Здесь три группы:

  * `lighthouse` (на одном хосте deb-1 с IP 81.19.136.70)
  * `vector` (на одном хосте el-1 с IP 81.19.136.48)
  * `clickhouse` (на том же хосте deb-1 с IP 81.19.136.70)

* **requirements.yml**

  ```yaml
  ---
  - src: git@github.com:Chika1703/vector-role.git
    scm: git
    version: "1.0.0"
    name: vector

  - src: cons ensys.lighthouse
    version: 0.0.13

  - src: git@github.com:AlexeySetevoi/ansible-clickhouse.git
    scm: git
    version: "1.13"
    name: clickhouse
  ```

  Устанавливает три роли:

  1. `vector` из моего репозитория
  2. `consensys.lighthouse` из Galaxy
  3. `clickhouse` из GitHub (v1.13)

* **site.yml**

  ```yaml
    ---
    - hosts: clickhouse
      roles:
        - clickhouse

    - hosts: vector
      roles:
        - vector

    - hosts: lighthouse
      roles:
        - consensys.lighthouse
  ```

  **Пояснения:**

  * Для каждой группы (`clickhouse`, `vector`, `lighthouse`) вызывается соответствующая `role`.
  * Роль `consensys.lighthouse` принимает необходимые переменные (`lighthouse_version` и т.д.) через секцию `vars` в playbook.

---

## Выполнение

1. **Создание `requirements.yml` и установка ролей**

   ```bash
   cd /root/chmoNya/playbook
   ansible-galaxy install -r requirements.yml -p roles
   ```

   Результат:

   * Папка `roles/vector` (из вашего репозитория)
   * Папка `roles/consensys.lighthouse` (из Galaxy)
   * Папка `roles/clickhouse` (из AlexeySetevoi/ansible-clickhouse)

## Скриншоты

* **Запуск playbook**
![Check Mode](https://github.com/Chika1703/ansible-playbook/blob/main/img/1.jpg)

---

## Заключение

В ходе выполнения домашнего задания:

* Созданы три роли:

  1. **ClickHouse** — внешняя роль [AlexeySetevoi/ansible-clickhouse](https://github.com/AlexeySetevoi/ansible-clickhouse).
  2. **Vector** — самостоятельная роль `vector-role`, опубликованная в репозитории [https://github.com/Chika1703/vector-role](https://github.com/Chika1703/vector-role).
  3. **LightHouse** — роль из Ansible Galaxy `consensys.lighthouse` (версия 0.0.13), позволяющая автоматически установить и настроить Lighthouse v12.3.0.

* Главный playbook переработан под использование новых ролей и протестирован.

* Задание выполнено:

  * Есть репозиторий с ролью Vector: [https://github.com/Chika1703/vector-role](https://github.com/Chika1703/vector-role)
  * Есть основной репозиторий с playbook и настройкой трёх ролей: [https://github.com/Chika1703/ansible-playbook](https://github.com/Chika1703/ansible-playbook)
