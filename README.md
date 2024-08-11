# VATS

vats - библиотека Ansible для автоматизации деплоя сервисов в Docker Swarm и настройки виртуалок.  
Поддерживает автоматическую подстановку секретов из HashiCorp Vault.

В папке examples - примеры использования.

## Установка и первичная настройка
Для Windows все эти действия проделывать в **WSL**, нативной Ansible на Windows нет.

### 1) Установить Ansible
[Установить Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)  
Вкратце:
```bash
# Установка
python3 -m pip install --user ansible

# Обновление
python3 -m pip install --upgrade --user ansible
```

### 2) Установить необходимые зависимости для vats
```
sudo apt install sshpass rsync
pip3 install hvac
``` 
Так же нужен установленный **Docker**

### 3) Установить (или обновить) vats
Установить (или обновить) библиотеку vats из файла `requirements.yaml`:
```bash
ansible-galaxy install -r requirements.yaml
```

Или сразу из репозитория:
```bash
ansible-galaxy collection install git@github.com:ariallas/vats.git,main
```

### 4) Добавить переменную окружения со своим токеном vault 
Для работы библиотеки в окружении должна быть определена переменная окружения `VAULT_TOKEN` с токеном к волту  
Чтобы получить свой токен:
- Зайти в Vault
- Нажать на свою иконку в правом верхнем углу
- Нажать **Copy token**

Добавьте в `~/.bashrc` строку c полученным токеном:
```bash
export VAULT_TOKEN=hvs.CAES...
```

Изменение применится после открытия нового окна терминала или команды: `source ~/.bashrc`

⚠ Время жизни токена - 30 дней, его нужно периодически обновлять.

## Состав
Роли:
- `init` - подключает стандартные роли для инициализации:
	- `secrets_from_vault` - получение секретов из волта
	- `first_contact` - автоматическое добавления SSH ключей инвентарных хостов в known_hosts localhost'а (чтобы не было интерактивного промпта при первом подключении)
	- `default_vars` - устанавливает значения глобальных переменных для проекта
- Деплой сервисов
	- `build_and_push_image` - сборка Docker образа и его заливка в Gitlab
	- `config_copy` - копирование файлов конфигурации (с возможностью шаблонизации), утилиты для создания Docker Swarm конфигов
	- `generic_deploy` - роль высокого уровня, использует все другие. Собирает образ, копирует файлы конфигурации и деплоит стек в Docker Swarm
	- `docker_login` - логин в Gitlab с использованием секретов `docker_user` и `docker_token`
- Настройка виртуалок
	- `install_docker` - установка и первичная настройка Docker
	- `docker_swarm` - инициализировать Docker Swarm на мастер и воркер нодах, установить лейблы нод
	- `ntp_setup` - настраивает синхронизацию времени с ntp сервером
	- `ssh_connection` - для использования в консоли, выводит строку для подключения к машине из файла инвентаря

## Доступ к машине по SSH
После подготовки возможно получить доступ к машинам при помощи команды (подставить корректный файл инвентаря):
```
ansible all --module-name include_role --args name=va.vats.ssh_connection -i <inventory_file>
```

## Секреты в Vault

Должно быть задано две переменных секретов:
```yaml
vault_path_inventory: secret-path/data/secret-key,inventory_secrets.yaml
vault_path_secrets_v2:
    - path: secret-path/data/postgres,postgres_test.yaml
      key: postgres
      format: yaml  # Возможные значения: yaml, string
```

Для секрета с **docker_login** и **docker_token** нужно указывать `key: vats`!

vault_path_inventory определяет путь к секретам для подключения к хостам, а vault_path_secrets_v2 - остальные секреты

Пример содержимого inventory_secrets.yaml:

```yaml
some-hostname: # Инвентарное имя хоста
  ansible_host: 10.0.0.1
  ansible_user: user
  ansible_password: password
  ansible_become_user: root
  ansible_become_password: root_password
  ansible_become_method: su
some-other-hostname:
  ansible_host: 10.0.0.2
  ...
```

Пример содержимого секрета `postgres_test.yaml`:

```yaml
postgres_host: "127.0.0.1"
postgres_port: 5432
...
```

Содержимое секрета (словарь или строка) помещается в переменную `secrets.postgres`, т.е. обращение к переменным производится так: `secrets.postgres.postgres_host`
