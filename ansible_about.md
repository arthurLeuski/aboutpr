# Отчёт по работе с Prometheus, Grafana и Ansible
## 1. Изучение Prometheus и Grafana
Первое знакомство с метриками
Разобрался, как работает связка Prometheus и Grafana. Основная идея такая:
я могу подготавливать метрики в приложениях, подключать библиотеки для экспорта, а Prometheus будет забирать эти метрики по указанным эндпоинтам. Далее данные можно визуализировать в Grafana.

## Пример конфига Prometheus
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'webapp'
    static_configs:
      - targets: ['webapp:5001']
```
## Что здесь происходит
scrape_interval: 15s — интервал между опросами метрик
job_name — название задачи, которую Prometheus будет мониторить
targets — адреса, откуда Prometheus забирает метрики

В данном примере собираются метрики:

самого Prometheus

нагрузки на систему через node-exporter

метрики моего приложения

## Дополнительное наблюдение
Оказалось, что дашборды можно не только собирать вручную в Grafana, но и хранить их в файлах. Это удобно для версионирования и переноса между окружениями.

## 2. Работа с Ansible
Первые шаги
Ansible понравился больше, чем ожидал. После установки нужно было:

добавить пользователей в sudo группы на всех виртуалках

прокинуть SSH‑ключи с хоста

подготовить inventory файл

Хостом выступает мой ноутбук, а управляемыми машинами — две виртуалки на сервере.

Inventory файл
```yaml
all:
  hosts:
    vm1:
      ansible_host: 192.168.31.86
      ansible_user: arthur
      ansible_become: yes

    vm2:
      ansible_host: 192.168.31.24
      ansible_user: devops
      ansible_become: yes

```
ansible_become: yes — использовать sudo при выполнении задач.

## Первые плейбуки
```yaml
- name: Создать папку
  hosts: vm1,vm2
  become: yes
  tasks:
    - name: Create empty file
      file:
        path: "/home/empty_file.txt"
        state: touch
```
Модуль file позволяет работать с файловой системой. Таких модулей много, и пока нет необходимости знать их все. Достаточно искать нужный модуль под конкретную задачу.

Переход к более сложной структуре
Сначала у меня был один большой плейбук, но позже я разбил его на роли.
Так стало удобнее поддерживать конфигурации и добавлять новые задачи.

## 3. Итог проделанной работы
За это время я:

развернул две виртуалки на сервере

подключился к ним через Ansible со своего ноутбука

настроил базовую автоматизацию через плейбуки

научился разбивать плейбуки на роли

разобрался с основами Prometheus и Grafana

настроил сбор метрик с нескольких источников

понял, как строить дашборды и как хранить их в файлах

Получился хороший фундамент для дальнейшего изучения DevOps инструментов.


# Нужно добавить инфомармацию про пелйбуки и их разбиение 
```yaml
- name: User and backup setup
  hosts: all
  become: yes

  vars:
    username: "backupuser"
    backup_dir: "/opt/backup"
    source_dir: "/var/www"
    cron_script: "/usr/local/bin/backup.sh"

  tasks:
    - name: Create user
      user:
        name: "{{ username }}"
        shell: /bin/bash

    - name: Create backup directory
      file:
        path: "{{ backup_dir }}"
        state: directory
        owner: "{{ username }}"
        group: "{{ username }}"
        mode: "0755"

    - name: Create backup script
      copy:
        dest: "{{ cron_script }}"
        mode: "0755"
        content: |
          #!/bin/bash
          tar -czf {{ backup_dir }}/backup_$(date +%F).tar.gz {{ source_dir }}

    - name: Add cron job (every 3 days)
      cron:
        name: "Backup every 3 days"
        user: "{{ username }}"
        job: "{{ cron_script }}"
        minute: "0"
        hour: "3"
        day: "*/3"
```

## Тут описал монолитый подход который дальше нужно было разбить , можно делать и так, но если хочу переиспользовать одну функцию плюсов мало
### А вот далее разбиваю первым делом плейбук на мини-плейбуки , на роли, и описываю единый плейбук в которой просто использую роли
```yaml
- name: Setup user, directory and backups
   hosts: all become: yes
   vars:
      username: "backupuser"
      backup_dir: "/opt/backup"
      source_dir: "/var/www"
  roles:
   - user
   - directory
   - backup
```
```yaml
- name: Create user
  user:
    name: "{{ username }}"
    shell: /bin/bash
```
```yaml
  - name: Create backup directory
    file:
      path: "{{ backup_dir }}"
      state: directory
      owner: "{{ username }}"
      group: "{{ username }}"
      mode: "0755"
```
```yaml
- name: Install backup script
  template:
    src: backup.sh.j2
    dest: /usr/local/bin/backup.sh
    mode: "0755"

- name: Add cron job (every 3 days)
  cron:
    name: "Backup every 3 days"
    user: "{{ username }}"
    job: "/usr/local/bin/backup.sh"
    minute: "0"
    hour: "3"
    day: "*/3"
```

