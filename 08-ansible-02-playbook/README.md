
# Домашнее задание к занятию 2 «Работа с Playbook»

## Подготовка к выполнению

1. * Необязательно. Изучите, что такое [ClickHouse](https://www.youtube.com/watch?v=fjTNS2zkeBs) и [Vector](https://www.youtube.com/watch?v=CgEhyffisLY).
2. Создайте свой публичный репозиторий на GitHub с произвольным именем или используйте старый.
3. Скачайте [Playbook](./playbook/) из репозитория с домашним заданием и перенесите его в свой репозиторий.
4. Подготовьте хосты в соответствии с группами из предподготовленного playbook.

## Основная часть

1. Подготовьте свой inventory-файл `prod.yml`.
2. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает [vector](https://vector.dev).
3. При создании tasks рекомендую использовать модули: `get_url`, `template`, `unarchive`, `file`.
4. Tasks должны: скачать дистрибутив нужной версии, выполнить распаковку в выбранную директорию, установить vector.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Подготовьте README.md-файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
```
---
#первый плей установка Clickhouse
- name: Install Clickhouse
  hosts: clickhouse
# Хендлер будет выполнен в конце плея и при условии его успешной отработки
  handlers:
#Рестарт\старт Clickhouse
    - name: Start clickhouse service
      become: true
#Модуль управления службами
      ansible.builtin.service:
        name: clickhouse-server
        state: restarted
#Рестарт\старт Vector
    - name: Start Vector service
      become: true
      ansible.builtin.service:
        name: vector
        state: restarted
#Рестарт\старт Vector
  tasks:
    - block:
#Загрузка дистрибутива
        - name: Get clickhouse distrib
#Загружает файлы с HTTP,HTTPS или FTP на узел
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/{{ item }}-{{ clickhouse_version }}.noarch.rpm"
            dest: "./{{ item }}-{{ clickhouse_version }}.rpm"
          with_items: "{{ clickhouse_packages }}"
      rescue:
        - name: Get clickhouse distrib
          ansible.builtin.get_url:
            url: "https://packages.clickhouse.com/rpm/stable/clickhouse-common-static-{{ clickhouse_version }}.x86_64.rpm"
            dest: "./clickhouse-common-static-{{ clickhouse_version }}.rpm"
#Установка загруженного пакета
    - name: Install clickhouse packages
      become: true
#Управление пакетами с помощью менеджера пакетов yum
      ansible.builtin.yum:
        name:
          - clickhouse-common-static-{{ clickhouse_version }}.rpm
          - clickhouse-client-{{ clickhouse_version }}.rpm
          - clickhouse-server-{{ clickhouse_version }}.rpm
#Запускает хендлер (Если таска не отработала он не запустится)
      notify: Start clickhouse service
    - name: Flush handlers
      meta: flush_handlers
    - name: Create database
#Модуль выполнения команд на цели, модуль принимает имя команды
      ansible.builtin.command: "clickhouse-client -q 'create database logs;'"
      register: create_db
      failed_when: create_db.rc != 0 and create_db.rc !=82
      changed_when: create_db.rc == 0
# Vector, выполнено по тому же алгоритму и с пременением тех же модулей (Скачивание, установка, перезагрузка(в случае уданой отработки предид>
- name: Vector installation
  hosts: vector
  tasks:
    - name: Install additional tools
      become: true
      ansible.builtin.yum:
        name:
          - "vim-enhanced"
          - "libstdc++"
          - "glibc"
        state: present
    - name: Downloading Vector distributives
      ansible.builtin.get_url:
        mode: 0644
        url: "https://packages.timber.io/vector/0.21.1/{{ vector_latest }}.rpm"
        dest: "/tmp/{{ vector_latest }}.rpm"
    - name: Install Vector packages
      become: true
      ansible.builtin.yum:
        name: "/tmp/{{ vector_latest }}.rpm"
        state: present
      notify: Start Vector service
...
```

11. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-02-playbook` на фиксирующий коммит, в ответ предоставьте ссылку на него.

---

### Как оформить решение задания

Выполненное домашнее задание пришлите в виде ссылки на .md-файл в вашем репозитории.

---
