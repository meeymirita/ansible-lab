# План лабы (из plan/ родительского репозитория, 11.10.2026)

> Общие правила (режимы «с нуля»/«продолжение», стандарт подробности «каждое слово объяснено») — в `plan/README.md` родительского репозитория `anitech-performance`.

# Ansible Lab (`ansible`)

| | |
|---|---|
| Режим | **продолжение Docker** (разворачивает его образ и compose) **или самостоятельная** (`starter/` содержит образ и `compose.yaml` CoffeeShop) |
| Размер | 10 сессий × ~4 ч = **~40 ч**, ~65 шагов |
| Сквозной проект | **Выкатка CoffeeShop на «серверы»**: 2–3 «сервера» (контейнеры с `sshd`), позже балансировщик и обновление без простоя |
| Версии | `ansible-core` актуальной ветки и коллекция `community.docker` (**сверьтесь**), Ubuntu LTS в «серверах», nginx 1.30 |
| Важно | реальный сервер владельца **не используется**; приватные SSH-ключи и пароль Vault — только локально и в `.gitignore` |

**Откуда темы:** разд. 8 (основы) → сессии 1–2; разд. 9 (playbook) → 3–6; разд. 10 (Vagrant) → 9 (как вариант); разд. 12 (продвинутое) → 7–9; разд. 13 (кластер) → 10; разд. 14 (итог) → 10.

### Сессия 1 · Азбука и стенд (~4 ч)
**Слова:** управление конфигурацией, push/pull, agentless, SSH, управляющий узел, управляемый узел, идемпотентность, YAML.
- 1.1 Проблема «сервер-снежинка»; инфраструктура как код; Ansible vs Chef/Puppet/Terraform (кратко)
- 1.2 Архитектура: control node → SSH → managed node, модули, как Ansible выполняет задачу (копирует модуль, запускает, удаляет)
- 1.3 Стенд: «серверы» как контейнеры с `sshd` в `compose.yaml` (Ubuntu, `openssh-server`, пользователь); **как ещё**: VM (Multipass/Lima/UTM — обзор)
- 1.4 SSH с нуля: `ssh-keygen -t ed25519` (флаги разобраны), публичный/приватный ключ, `authorized_keys`, `known_hosts`, вход без пароля
- 1.5 Установка Ansible (`brew`/`pip`), проверка `ansible --version`, конфиг `ansible.cfg`
- 1.6 Режимы «с нуля» / «продолжение»
- 🔨 `Permission denied (publickey)` — по шагам
- **Приёмка:** `ssh` на каждый «сервер» без пароля

### Сессия 2 · Inventory и ad-hoc (~4 ч)
**Слова:** inventory, группа, `host_vars`, `group_vars`, модуль, ad-hoc, `become`.
- 2.1 Inventory INI и YAML, группы и вложенные группы, `all`/`ungrouped`
- 2.2 `ansible-inventory --list/--graph`; паттерны хостов (`web:&prod`, `!db`)
- 2.3 `ansible all -m ping`; как читать ответ (`SUCCESS`, `changed`, `ping/pong`)
- 2.4 Ad-hoc: `command`, `shell`, `copy`, `file`, `apt`, `service`, `setup`; чем `command` отличается от `shell`
- 2.5 `become`, `sudo` без пароля в «серверах», `-K`
- 2.6 `host_vars`/`group_vars`, порядок приоритета (первое знакомство)
- 🔨 «UNREACHABLE»: хост, порт, ключ, `known_hosts`
- **Приёмка:** ad-hoc ставит пакет на группу и показывает результат

### Сессия 3 · Первый playbook и идемпотентность (~4 ч)
**Слова:** playbook, play, task, модуль, состояние (`state`), `changed`/`ok`, `--check`, `--diff`.
- 3.1 Структура: `hosts`, `tasks`, `name`, отступы YAML
- 3.2 Модули `apt`, `file`, `copy`, `user`, `service`, `lineinfile` и «декларативный» подход
- 3.3 Запуск: `ansible-playbook`, вывод `PLAY RECAP` построчно
- 3.4 Идемпотентность: запуск дважды — `changed=0`; чем плох `command`/`shell`
- 3.5 `--check`, `--diff`, `--list-tasks`, `--start-at-task`, `-v…-vvv`
- 3.6 Проект: playbook ставит nginx и страницу на «серверы»
- 🔨 playbook, который не идемпотентен
- **Приёмка:** второй запуск даёт `changed=0`

### Сессия 4 · Переменные и факты (~4 ч)
**Слова:** переменная, факт, `register`, приоритет переменных, `debug`, `set_fact`.
- 4.1 Где определять переменные (play, файл, inventory, `-e`) и приоритеты — таблица
- 4.2 Факты `gather_facts`, `ansible_facts`, `setup`, `ansible_os_family`
- 4.3 `register` и `debug`, чтение результата модуля
- 4.4 `vars_files`, `vars_prompt`, `set_fact`, магические переменные (`inventory_hostname`, `groups`, `hostvars`)
- 4.5 Отладка: `debug`, `-vvv`, `ansible-playbook --step`
- 🔨 «переменная не определена» — найти, где потеряна
- **Приёмка:** playbook адаптируется под ОС по фактам

### Сессия 5 · Логика, ошибки, handlers (~4 ч)
**Слова:** условие `when`, цикл `loop`, handler, `notify`, `block`/`rescue`/`always`, `async`.
- 5.1 `when`: условия, `and/or`, `is defined`, по фактам
- 5.2 `loop`, `with_items` (устарело), `loop_control`
- 5.3 Handlers: `notify`, `flush_handlers`, когда срабатывают
- 5.4 `block/rescue/always`, `failed_when`, `changed_when`, `ignore_errors`
- 5.5 `async`/`poll`, долгие задачи
- 5.6 `tags`, `--tags`/`--skip-tags`
- 🔨 handler не срабатывает (задача не `changed`)
- **Приёмка:** конфиг nginx меняется → nginx перезапускается ровно один раз

### Сессия 6 · Jinja2 и шаблоны (~4 ч)
**Слова:** шаблон, Jinja2, фильтр, lookup, `template`, `ansible-lint`.
- 6.1 Jinja2: `{{ }}`, `{% %}`, `{# #}`, условия и циклы в шаблонах
- 6.2 Фильтры: `default`, `join`, `map`, `select`, `to_json`, `regex_replace`
- 6.3 Lookup-плагины: `file`, `env`, `pipe`, `password`
- 6.4 Шаблон `nginx.conf.j2` с переменными и циклом по сайтам
- 6.5 `ansible-lint`: установка, правила, исправление замечаний
- 🔨 лишний пробел/отступ в шаблоне ломает конфиг
- **Приёмка:** `ansible-lint` без замечаний на проекте

### Сессия 7 · Роли и Galaxy (~4 ч)
**Слова:** роль, `defaults`, `vars`, `meta`, коллекция, Galaxy, `requirements.yml`.
- 7.1 Зачем роли; структура каталога (`tasks/handlers/templates/defaults/files/meta`)
- 7.2 Роль `docker_host`: установка Docker на «сервер»
- 7.3 Роль `nginx` с шаблонами
- 7.4 Galaxy и коллекции: `ansible-galaxy install`, `requirements.yml`, `community.docker`
- 7.5 Зависимости ролей, повторное использование, порядок `pre_tasks`/`roles`/`tasks`
- 🔨 роль «не видит» переменную из `defaults`
- **Приёмка:** `site.yml` из ролей, повторный запуск `changed=0`

### Сессия 8 · Секреты: Vault и безопасность (~4 ч)
**Слова:** Vault, шифрование, `vault-id`, `no_log`, секрет, принцип наименьших привилегий.
- 8.1 Зачем шифровать; что **нельзя** класть в git (напоминание: пароль Vault — вне репозитория)
- 8.2 `ansible-vault create/edit/view/encrypt/decrypt/rekey`
- 8.3 Использование в playbook: `vars_files`, `--ask-vault-pass`, файл пароля вне репо, `vault-id`
- 8.4 `no_log: true` и утечки в выводе
- 8.5 Безопасность SSH и `sudo`: отдельный пользователь, ограничение прав
- 🔨 playbook пишет пароль в лог
- **Приёмка:** пароль БД зашифрован и подставляется в контейнер

### Сессия 9 · Docker через Ansible и варианты стенда (~4 ч)
**Слова:** коллекция `community.docker`, образ, GHCR, compose-модуль, VM, Vagrant.
- 9.1 Сборка образа модулем `docker_image`; вход в GHCR (токен из переменной)
- 9.2 Запуск контейнеров `docker_container`, сети и тома модулями
- 9.3 Развёртывание compose-проекта (`docker_compose_v2`)
- 9.4 Swarm через Ansible: `docker_swarm`, `docker_stack` (если пройдена часть I сессий 11–12)
- 9.5 **Вариант стенда: виртуальные машины** — Multipass/Lima; **Vagrant + VirtualBox** на Apple Silicon (сверьтесь, не проверено), Vagrantfile, `vagrant up/ssh/destroy`, как отдать VM Ansible
- 🔨 «образ не найден» на втором узле
- **Приёмка:** CoffeeShop развёрнут на «серверах» одним playbook

### Сессия 10 · Кластер, обновление без простоя, итог (~4 ч)
**Слова:** `delegate_to`, `local_action`, `serial`, `pre_tasks`/`post_tasks`, балансировка, rolling update.
- 10.1 nginx как балансировщик над двумя узлами (шаблон upstream)
- 10.2 `delegate_to` / `local_action`, `run_once`
- 10.3 `pre_tasks`/`post_tasks`, проверка готовности (`uri`, `wait_for`)
- 10.4 `serial: 1`: обновление по одному узлу, **вывод узла из балансировки** на время обновления
- 10.5 Итоговый проект: `site.yml` разворачивает всё с нуля; смена тега образа — обновление без простоя
- 🔨 **Production Hell** без подсказок: 5 поломок
- **Приёмка:** `ansible-playbook site.yml` на чистом стенде → сайт; повторно `changed=0`; обновление не роняет сайт

**Справка:** симптом → причина → что делать (`UNREACHABLE`, `Failed to connect to the host via ssh`, `MODULE FAILURE`, `Missing sudo password`, `vault password`…), команды, глоссарий, 50 вопросов, «что дальше».

---

