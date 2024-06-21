# Домашняя работа: Ansible

Цель работы: Освоить инструмент Ansible

Что нужно сделать?

Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере, используя Ansible необходимо развернуть nginx со следующими условиями:

- необходимо использовать модуль yum/apt
- конфигурационный файлы должны быть взяты из шаблона jinja2 с переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible

*  Сделать все это с использованием Ansible роли

# Выполнение

## Создаём виртуальную машину

Создаю в домашней директории Vagrantfile, в тело данного файла копирую содержимое из прилагаемой методички.
 
Собираю стенд командой:

``` [nur@test Ansible]$ vagrant up ```

Подключаюсь к стенду:

```[nur@test Ansible]$ vagrant ssh
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-102-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Fri Jun 21 05:00:28 PM UTC 2024

  System load:  1.90380859375      Processes:             160
  Usage of /:   12.0% of 30.34GB   Users logged in:       0
  Memory usage: 24%                IPv4 address for eth0: 10.0.2.15
  Swap usage:   0%                 IPv4 address for eth1: 192.168.56.15


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Mon Jun 17 16:48:51 2024 from 10.0.2.2
vagrant@nginx:~$
```


Получаем параметры ssh и проверяем управление Ansible - созданный стенд:
``` [nur@test Ansible]$ vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/nur/hw-2/Ansible/.vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
  PubkeyAcceptedKeyTypes +ssh-rsa
  HostKeyAlgorithms +ssh-rsa
```
Создаём директорию "staging" и в ней фаил "host" со следующим содержимым:
``` 
[web]

nginx ansible_host=127.0.0.1 ansible_port=2222  ansible_user=vagrant ansible_private_key_file=/home/nur/hw-2/Ansible/.vagrant/machines/nginx/virtualbox/private_key

```

``` [nur@test Ansible]$ ansible nginx -i staging/hosts -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}

```
Создаем фаил "ansible.cfg" cо следующим содержимым:
``` 
[defaults]

inventory = staging/hosts

remote_user = vagrant

host_key_checking = False

retry_files_enabled = False
```
Из файла "hosts" убираем информацию о пользователе и проверяем управление без inventory - файла:

```
[nur@test Ansible]$ ansible nginx  -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    }, 
    "changed": false, 
    "ping": "pong"
}
```
Подключение успешно.

## Используем модуль yum/apt и переменные для прослушивания порта 8080

Создаем playbook - фаил "nginx.yml" cо следующим содержимым:
```
 name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks: 
    - name: update
      apt:
        update_cache=yes
      tags:
        - update apt

    - name: NGINX | Install NGINX
      apt:
        name: nginx
        state: latest
      tags:
        -nginx-package
```
## Получаем конфигурационные файлы из шаблона jinja2 с переменными

Дополняем playbook - фаил "nginx.yml" cледующей секцией:

```
- name: NGINX | Create NGINX config file from template
      template:
        src: files/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
```
## Используем notify и добавляем режим enabled nginx в systemd

Дополняем playbook - фаил "nginx.yml" cледующей секцией:

```
 notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```
Запускаем результирующий фаил "nginx.yml":

```
[nur@test Ansible]$ ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] *************************************

TASK [Gathering Facts] *********************************************************
ok: [nginx]

TASK [update] ******************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX] ***************************************************
changed: [nginx]

TASK [NGINX | Create NGINX config file from template] **************************
changed: [nginx]

RUNNING HANDLER [reload nginx] *************************************************
changed: [nginx]

PLAY RECAP *********************************************************************
nginx                      : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```   
playbook - фаил отработал.

*  Применяем  Ansible роли

Создаем в текущем каталоге директорию "roles", переходим в неё и создаём роль nginx-2 :
```
[nur@test roles]$ ansible-galaxy init nginx-2
- Role nginx-2 was created successfully
```
Переходим в директорию nginx и наблюдаем следующую структуру каталогов:

```
[nur@test roles]$ cd nginx-2
[nur@test nginx-2]$ ls
defaults  files  handlers  meta  README.md  tasks  templates  tests  vars

```
Создадим в  директории "roles" фаил "nginx.yml" cо следующим содержимым:
```
---
- hosts: all
  become: true
  roles:
    - nginx-2
```
В директорию files кладём шаблон "nginx.conf.j2".

В директории "handlers" дополняем фаил "main.yml" следующим содержимым:
```
---
- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes

- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```
В директории "tasks" дополняем фаил "main.yml" cледующим содержимым:
```
---
- name: update
  apt:
    update_cache=yes
  tags:
    - update apt

- name: NGINX | Install NGINX
  apt:
    name: nginx
    state: latest
  tags:
    -nginx-package

- name: NGINX | Create NGINX config file from template
  template:
    src: files/nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx
  tags:
    - nginx-configuration
```
В директории "vars" дополняем фаил "main.yml" cледующим содержимым:
```
---
nginx_listen_port: 8080
```
Пересоздадим чистый стенд и запускаем на исполнение роль nginx-2:
```
 [nur@test Ansible]$ ansible-playbook nginx.yml

PLAY [NGINX | Install and configure NGINX] ***********************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************
ok: [nginx]

TASK [update] ****************************************************************************************************************************************
ok: [nginx]

TASK [NGINX | Install NGINX] *************************************************************************************************************************
changed: [nginx]

TASK [NGINX | Create NGINX config file from template] ************************************************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] ***********************************************************************************************************************
changed: [nginx]

PLAY RECAP *******************************************************************************************************************************************
nginx                      : ok=5    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
Роль отработала.

Полключаемся к стенду и проверяем доступность сайта:
```
  vagrant@nginx:~$ curl http://192.168.56.15:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```
Сайт доступен.
