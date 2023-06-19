# Домашнее задание к занятию 3 «Использование Yandex Cloud»

## Подготовка к выполнению

1. Подготовьте в Yandex Cloud три хоста: для `clickhouse`, для `vector` и для `lighthouse`.
2. Репозиторий LightHouse находится [по ссылке](https://github.com/VKCOM/lighthouse).

## Основная часть

1. Допишите playbook: нужно сделать ещё один play, который устанавливает и настраивает LightHouse.
2. При создании tasks рекомендую использовать модули: `get_url`, `template`, `yum`, `apt`.
3. Tasks должны: скачать статику LightHouse, установить Nginx или любой другой веб-сервер, настроить его конфиг для открытия LightHouse, запустить веб-сервер.
4. Подготовьте свой inventory-файл `prod.yml`.
5. Запустите `ansible-lint site.yml` и исправьте ошибки, если они есть.
6. Попробуйте запустить playbook на этом окружении с флагом `--check`.
7. Запустите playbook на `prod.yml` окружении с флагом `--diff`. Убедитесь, что изменения на системе произведены.
8. Повторно запустите playbook с флагом `--diff` и убедитесь, что playbook идемпотентен.
9. Подготовьте README.md-файл по своему playbook. В нём должно быть описано: что делает playbook, какие у него есть параметры и теги.
```
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
  tags:
    - clickhouse
      
# Vector, выполнено по тому же алгоритму и с пременением тех же модулей (Скачивание, установка, перезагрузка)
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
  tags:
    - vector
# Установка lighthouse, использованны те же модули что идля предидущих плеев
- name: Install lighthouse 
  hosts: lighthouse
  handlers: # Хэндлеры для запуска и перезагрузки nginx в случае удачной отработки тасок
    - name: start-nginx
      become: true
      ansible.builtin.command: "nginx"
      when: not ansible_check_mode # Пропуск задач или игнорирование ошибок в режиме проверки
    - name: reload-nginx
      become: true
      ansible.builtin.command: "nginx -s reload"
      when: not ansible_check_mode # Пропуск задач или игнорирование ошибок в режиме проверки
  tasks: 
    - name: Instal epel-release # установка пакета epel-release для nginx
      become: true
      ansible.builtin.yum:
        name: epel-release
    - name: Install nginx # установка nginx
      become: true
      ansible.builtin.yum:
        name: nginx
      notify: start-nginx
    - name: Install git # установка гита для установки lighthouse
      become: true
      ansible.builtin.yum:
        name: git
    - name: Download lighthouse # скачивание lighthouse в каталог
      become: true
      ansible.builtin.git:
        repo: 'https://github.com/VKCOM/lighthouse.git'
        dest: /var/www/lighthouse
        update: no
    - name: Fix owner and mode # установка владельца и прав
      become: true
      ansible.builtin.file:
        path: /var/www/lighthouse
        state: directory
        mode: '0755'
        owner: ivan
    - name: nginx config # копирование конфига lighthouse в конфиг nginx
      become: true
      ansible.builtin.template:
        src: template/lighthouse.j2
        dest: /etc/nginx/nginx.conf
        owner: ivan
      notify: reload-nginx
  tags:
    - lighthouse
...

Использованны тэги:
- clickhouse
- vector
- lighthouse

11. Готовый playbook выложите в свой репозиторий, поставьте тег `08-ansible-03-yandex` на фиксирующий коммит, в ответ предоставьте ссылку на него.

---

### Как оформить решение задания

Выполненное домашнее задание пришлите в виде ссылки на .md-файл в вашем репозитории.

---
