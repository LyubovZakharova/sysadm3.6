1)
A.. Устанавливаем нужную версию:
Скачиваем:
:~$ wget https://github.com/prometheus/node_exporter/releases/download/v1.3.0/node_exporter-1.3.0.linux-amd64.tar.gz
Распаковываем:
:~$ tar xvfz node_exporter-1.3.0.linux-amd64.tar.gz
Переходим в директорию:
:~$ cd node_exporter-1.3.0.linux-amd64
Проверяем запуск:
:~$ ./node_exporter
curl localhost:9100/metrics
Переместим node_exporter из домашней директории в /opt в соответствии с рекомендациями:
:~$ sudo mv node_exporter-1.3.0.linux-amd64 /opt

B.. Создадим внешний файл для добавления опций:
:~$ sudo vim /etc/default/node_exporter
Создадим юнит-файл для systemd:
:~$ sudo vim /etc/systemd/system/node_exporter.service
Содержание:
[Unit]
Description=Node Exporter

[Service]
EnvironmentFile=/etc/default/node_exporter
ExecStart=/opt/node_exporter-1.3.0.linux-amd64/node_exporter $OPTIONS

[Install]
WantedBy=multi-user.target
C.. Проверяем корректность процесса:
systemctl status node_exporter Получаем сообщение об ошибке:
Active: failed (Result: exit-code) since Fri 2021-12-10 21:45:53 UTC; 1min 16s ago
...
Dec 10 21:45:53 vagrant node_exporter[14658]: ts=2021-12-10T21:45:53.853Z caller=node_exporter.go:202 level=error err="listen tcp :9100: bind: address already in use
Подозреваем, что это продолжает работать запущенный node_exporter. И точно:
:~$ ps aux | grep node_exporter
vagrant    12771  0.0  1.5 717372 15460 pts/1    Sl+  Dec10   0:11 ./node_exporter
vagrant    14708  0.0  0.0   9032   724 pts/0    R+   21:49   0:00 grep --color=auto node_exporter
:~$ kill 12771 
Убиваем старый процесс,перезапускаем сервис,проверяем порт:
:~$ sudo systemctl restart node_exporter
:~$ sudo lsof -i:9100
COMMAND     PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
node_expo 14742 root    3u  IPv6 135995      0t0  TCP *:9100 (LISTEN)
ВСЕ ЗАПУСТИЛОСЬ.
D.. Перезагружаем систему sudo reboot. Проверяем systemctl status node_exporter - Active: active (running).
Проверяем stop и start

:~$ sudo systemctl stop node_exporter
:~$ sudo systemctl status node_exporter
● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: enabled)
     Active: inactive (dead) since Fri 2021-12-10 22:28:19 UTC; 8s ago
    Process: 599 ExecStart=/opt/node_exporter-1.3.0.linux-amd64/node_exporter $OPTIONS (code=killed, signal=TERM)
   Main PID: 599 (code=killed, signal=TERM)

[...]
Dec 10 22:28:19 vagrant systemd[1]: Stopping Node Exporter...
Dec 10 22:28:19 vagrant systemd[1]: node_exporter.service: Succeeded.
Dec 10 22:28:19 vagrant systemd[1]: Stopped Node Exporter.
:~$ sudo systemctl start node_exporter
:~$ sudo systemctl status node_exporter
● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2021-12-10 22:28:49 UTC; 2s ago
   Main PID: 1027 (node_exporter)
      Tasks: 4 (limit: 1071)
     Memory: 2.4M
     CGroup: /system.slice/node_exporter.service
             └─1027 /opt/node_exporter-1.3.0.linux-amd64/node_exporter

[...]
Dec 10 22:28:49 vagrant node_exporter[1027]: ts=2021-12-10F22:28:49.109Z caller=node_exporter.go:199 level=info msg="Listening on" address=:9100
Dec 10 22:28:49 vagrant node_exporter[1027]: ts=2021-12-10F22:28:49.110Z caller=tls_config.go:195 level=info msg="TLS is disabled." http2=false

2)
CPU:
    node_cpu_seconds_total{cpu="0",mode="idle"} 2238.49
    node_cpu_seconds_total{cpu="0",mode="system"} 16.72
    node_cpu_seconds_total{cpu="0",mode="user"} 6.86
    process_cpu_seconds_total
    
Memory:
    node_memory_MemAvailable_bytes 
    node_memory_MemFree_bytes
    
Disk(если несколько дисков то для каждого):
    node_disk_io_time_seconds_total{device="sda"} 
    node_disk_read_bytes_total{device="sda"} 
    node_disk_read_time_seconds_total{device="sda"} 
    node_disk_write_time_seconds_total{device="sda"}
    
Network(так же для каждого активного адаптера):
    node_network_receive_errs_total{device="eth0"} 
    node_network_receive_bytes_total{device="eth0"} 
    node_network_transmit_bytes_total{device="eth0"}
    node_network_transmit_errs_total{device="eth0"}

3) см. (prt sc - 3.)

4) см. (prt sc - 4.)

5) Параметр fs.nr_open - максимальное число файловых дескрипторов, который может открыть процесс.
ulimit -n - просмотреть существующий лимит  см.(prt sc - 5.) 

6) 
:~# ps -e | grep sleep
2020 pts/2    00:00:00 sleep
:~# nsenter --target 2020 --pid --mount
:~# ps
    PID TTY          TIME CMD
    2 pts/1    00:00:00 bash
   11 pts/1    00:00:00 ps
7)
:(){ :|:& };: - это bash-функция, которая запускает процесс, рекурсивно вызывающий клоны самого себя.
Используются для тестирования процессных ограничений, которые задаются в /etc/security/limits.conf.
: - функция передает себя через пайп себе же.
Функционал, должен быть такой:
[ 3099.973235] cgroup: fork rejected by pids controller in /user.slice/user-1000.slice/session-4.scope
[ 3103.171819] cgroup: fork rejected by pids controller in /user.slice/user-1000.slice/session-11.scope
Система на основании этих файлов,в пользовательской зоне ресурсов, имеет определенное ограничение на создаваемые ресурсы 
и,соответсвенно,при превышении -начинает блокировать создание числа .
Если установить ulimit -u 50 - число процессов будет ограничено 50 для пользоователя. 