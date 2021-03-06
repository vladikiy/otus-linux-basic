# Домашнее задание №7

Снять дамп обращения к веб-серверу, проанализировать пакеты (начиная с первого). Описать на примере снятого дампа, как устанавливается сессия TCP. Настроить в iptables policy DROP, прописать разрешения только на нужные порты и протоколы.

Цель: в результате выполнения ДЗ вы получите базовые навыки работы с iptables и tcpdump.

В данном задании тренируются навыки:

- анализ пакетов данных;
- фильтрация траффика;
- восстановление конфигурации фильтрации траффика после перезагрузки ОС.

Необходимо:

- собрать дамп пакетов с определенного IP и проанализовать их;
- проанализовать нужные порты назначения и адреса источников для дальнейшей фильтрации траффика;
- настроить разрешения в iptables;
- сделать автовосстановление правил фильтрации после перезагрузки ОС.

# Ход работы

Работа выполняется на CentOS 7.

## Установка tcpdump

Устанавливаем `tcpdump` из одноименного пакета:

```bash
yum install tcpdump
```
## Демонстрация установки TCP-сессии

Для демонстрации процесса установки TCP-сессии соберем входящие и исходящие пакеты с порта 80: ` tcpdump -nnn port 80`. В дампе идет установка
одного соединения.

```
# Входящий пакет с флагом SYN
16:22:57.957660 IP 10.100.102.4.46022 > 10.100.102.10.80: Flags [S], seq 3733233394, win 64240, options [mss 1460,sackOK,TS val 1111289652 ecr 0,nop,wscale 7], length 0

# Исходящий пакет с флагами SYN+ACK, подтверждающий прием SYN
16:22:57.957714 IP 10.100.102.10.80 > 10.100.102.4.46022: Flags [S.], seq 2658170832, ack 3733233395, win 28960, options [mss 1460,sackOK,TS val 251288941 ecr 1111289652,nop,wscale 7], length 0

# Входящий пакет с флагом ACK, TCP-сессия установлена
16:22:57.957768 IP 10.100.102.4.46022 > 10.100.102.10.80: Flags [.], ack 1, win 502, options [nop,nop,TS val 1111289652 ecr 251288941], length 0
```
С этого момента TCP-сессия установлена и возможна передача данных между клиентом и сервером по протоколу TCP/IP.

## Сбор дампа пакетов с определенного IP

Соберем 20 пакетов, принятых с адреса `10.100.102.4` и запишем в файл для дальнейшего анализа:

```bash
tcpdump -i ens33 'src 10.100.102.4' -c 20 -w packets.pcap
```
Во время сбора дампа несколько раз откроем страницу веб-сервера с компьютера с адресом 10.100.102.4.

## Анализ сохраненного дампа

Загрузим сформинованный в ранее дамп для анализа. Используем флаги для вывода адресов и портов в цифровом виде и максимально подробной информации о пакете, для удобства просмотра используем утилиту `less`:

```bash
tcpdump -nnvvv -r packets.pcap | less
```

Фрагмент дампа:

```
16:45:29.211523 IP (tos 0x0, ttl 64, id 4504, offset 0, flags [DF], proto TCP (6), length 52)
    10.100.102.4.42832 > 10.100.102.10.22: Flags [.], cksum 0xfd31 (correct), seq 2403720295, ack 3112627136, win 1453, options [nop,nop,TS val 1112640905 ecr 252640195], length 0
16:45:31.170468 IP (tos 0x0, ttl 64, id 33532, offset 0, flags [DF], proto TCP (6), length 474)
    10.100.102.4.46030 > 10.100.102.10.80: Flags [P.], cksum 0xca9f (correct), seq 1540306515:1540306937, ack 3471688435, win 501, options [nop,nop,TS val 1112642864 ecr 252638995], length 422: HTTP, length: 422
        GET / HTTP/1.1
        Host: 10.100.102.10
        User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0
        Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
        Accept-Language: en-US,en;q=0.5
        Accept-Encoding: gzip, deflate
        Connection: keep-alive
        Upgrade-Insecure-Requests: 1
        If-Modified-Since: Mon, 06 Jun 2022 10:49:57 GMT
        If-None-Match: "1c-5e0c53ba1c4e2"
```
Согласно дампу в 16:45:29.211523 по протоколу TCP/IP с адреса `10.100.102.4` и порта 42832 на адрес `10.100.102.10` порт 22 (SSH) пришел пакет с флагом "не фрагментировать" общей длиной 52 байта и длиной данных 0 байт.

В 16:45:31.170468 по протоколу TCP/IP с адреса `10.100.102.4` и порта 46030 на адрес `10.100.102.10` порт 80 пришел пакет общей длинной 474 байт, в том числе 422 байт данных протокола HTTP.

HTTP-запрос содержит следующую информцию:

1. Запрос делается методом GET к корню веб-сайта по протоколу HTTP версии 1.1.
2. Запрос выполняется на IP адрес (хост) `10.100.102.10`.
3. Для запроса используется браузер, идентифицирующий себя как `Mozilla/5.0 (X11; Linux x86_64; rv:102.0) Gecko/20100101 Firefox/102.0`.
4. В ответ на запрос ожидаются следующие типы содержимого: `text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8`.
5. Предпочитаемые языки в порядке уменьшения приоритета: `en-US,en;q=0.5`.
6. Поддерживаемые способы сжатия: `gzip, deflate`.
7. Указана опция сохранять соединение после отправки ответа.
8. Указана опция перейти на защищенный протокол, если поддерживается.
9. Клиент уже имеет в кэше данные, актуальные на `Mon, 06 Jun 2022 10:49:57 GMT`, сервер должен вернуь содержимое только если оно новее.
10. Клиент уже имеет в кэше данные с хэшем ` "1c-5e0c53ba1c4e2`, сервер должен вернуть содержимое только если его хэш отличен от указанного.

Таким образом, трафик поступает на порты 22 и 80 сервера. Соберем список всех портов, на которые поступали пакеты:

```bash
tcpdump -nnq -r packets.pcap | grep -P " > 10.100.102.10.(\d+)" -o | cut -d. -f 5 | sort -n | uniq
```
Полученный список совпадает с результатами анализа дампа, входящие пакеты приходят только на 22 и 80 порт.

## Настройка фильтрации трафика

Выполним настройку фильтрации трафика с помощью `iptables`:

```bash
yum install iptables-services
# разрешаем входящие подключения по loopback-интерфейсу
iptables -A INPUT -i lo -j ACCEPT
# разрешаем уже установленные соединения
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# отклоняем пакеты с ошибками
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP 
# разрешаем входящие подключения по SSH
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
# разрешаем входящие подключения по HTTP
iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
# разрешаем подключения к MySQL только по локальной сети
iptables -A INPUT --source 10.100.102.0/24 -p tcp --dport 3306 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
# все остальные входящие пакеты отклоняем
iptables -P INPUT DROP
# все исходящие соединения разрешаем
iptables -P OUTPUT ACCEPT
# запускаем iptables
systemctl enable iptables && systemctl start iptables
```

## Сохранение настроек iptables

Выполненные в прошлом разделе настройки будут активны до перезагрузки ОС. Чтобы они применялись автоматически при загрузке ОС необходимо их сохранить в файл конфигурации. Сделать это можно, например, следующей командой.

```bash
# сохраняем настройки
service iptables save
```

При этом настройки будут сохранены в файл `/etc/sysconfig/iptables`, из которого и будут считаны при загрузке ОС.

Таким образом, в рамках данной работы был выполнен сбор дампа входящих пакетов, проанализированы некоторые из пакетов, выявлены службы, к котором необходим доступ извне и настроены соответствующие правила фильтрации.
