# DNS- настройка и обслуживание.

Задание:

1. Взять стенд https://github.com/erlong15/vagrant-bind 

 - добавить еще один сервер client2;

 - завести в зоне dns.lab имена:

   - web1 смотрит на клиент1;

   - web2 смотрит на клиент2;

 - завести еще одну зону newdns.lab;

 - завести в ней запись:

   - www - смотрит на обоих клиентов.

2. настроить split-dns

 - клиент1 видит обе зоны, но в зоне dns.lab только web1;

 - клиент2 видит только dns.lab.


Цель:

Создать домашнюю сетевую лабораторию. Изучить основы DNS, научиться работать с технологией Split-DNS в Linux-based системах.

## Развертывание стенда для демострации настройки DNS.

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и четырех виртуальных машин `centos/7`.

Vagranfile описывает создание 4 виртуальных машин на CentOS 7, каждой машине будет выделено по 256 МБ ОЗУ. В начале файла есть модуль, который отвечает за настройку ВМ с помощью Ansible.

Машины с именами: `ns01` и `ns02` выполняют роль серверов DNS (master и slave соответственно).

Машины с именами: `client` и `client2` выполняют роль клиентов.

Разворачиваем инфраструктуру в Vagrant исключительно через Ansible.

В представленном стенде я реализовал выполнение второй части ДЗ, т.к. она является продолжение (усложнением) первой и отличается только наполнение файлов конфигураций серверов.

Все коментарии по каждому блоку указаны в тексте Playbook - `playbook.yml`.

Все файлы необходимые для проигрывания `playbook.yml` и сам файл Playbook необходимо поместить в каталог  `provisioning` с Vagranfile.

Рассмотрим требуемые нам файлы:

 - playbook.yml — это Ansible-playbook, в котором содержатся инструкции по настройке    нашего стенда;

 - client-motd — файл, содержимое которого будет появляться перед пользователем, который подключился по SSH;

 - named.ddns.lab и named.dns.lab — файлы описания зон ddns.lab и dns.lab соответсвенно;

 - master-named.conf и slave-named.conf — конфигурационные файлы, в которых хранятся настройки DNS-сервера;

 - client-resolv.conf и servers-resolv.conf — файлы, в которых содержатся IP-адреса DNS-серверов.


Выполняем установку стенда:

```
vagrant up
```

## Результат работы

### Проверка на client:
```
root@root-ubuntu:/home/roman/DNS# vagrant ssh client
Last login: Thu May 25 03:49:50 2023 from 10.0.2.2
### Welcome to the DNS lab! ###

- Use this client to test the enviroment, with dig or nslookup.
    dig @192.168.50.10 ns01.dns.lab
    dig @192.168.50.11 -x 192.168.50.10

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

Enjoy!
[vagrant@client ~]$ ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.022 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.068 ms
64 bytes from client (192.168.50.15): icmp_seq=3 ttl=64 time=0.066 ms
^C
--- www.newdns.lab ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 0.022/0.052/0.068/0.021 ms
[vagrant@client ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.037 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.067 ms
64 bytes from client (192.168.50.15): icmp_seq=3 ttl=64 time=0.070 ms
64 bytes from client (192.168.50.15): icmp_seq=4 ttl=64 time=0.086 ms
^C
--- web1.dns.lab ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 0.037/0.065/0.086/0.017 ms
[vagrant@client ~]$ ping web2.dns.lab
ping: web2.dns.lab: Name or service not known
```
### Проверка на client2:

```
root@root-ubuntu:/home/roman/DNS# vagrant ssh client2
Last login: Thu May 25 03:51:19 2023 from 10.0.2.2
### Welcome to the DNS lab! ###

- Use this client to test the enviroment, with dig or nslookup.
    dig @192.168.50.10 ns01.dns.lab
    dig @192.168.50.11 -x 192.168.50.10

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

Enjoy!
[vagrant@client2 ~]$ ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
[vagrant@client2 ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=2.85 ms
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=2 ttl=64 time=1.06 ms
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=3 ttl=64 time=1.08 ms
^C
--- web1.dns.lab ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2007ms
rtt min/avg/max/mdev = 1.065/1.669/2.859/0.841 ms
[vagrant@client2 ~]$ ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client2 (192.168.50.16): icmp_seq=1 ttl=64 time=0.039 ms
64 bytes from client2 (192.168.50.16): icmp_seq=2 ttl=64 time=0.112 ms
64 bytes from client2 (192.168.50.16): icmp_seq=3 ttl=64 time=0.067 ms
64 bytes from client2 (192.168.50.16): icmp_seq=4 ttl=64 time=0.062 ms
^C
--- web2.dns.lab ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 0.039/0.070/0.112/0.026 ms
```