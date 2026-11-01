<!--
Бланк звіту до лабораторної роботи «DNS».
Скопіювати цей файл у власний публічний репозиторій звіту як README.md і заповнити.
Коментарі такого вигляду в GitHub не відображаються; видаляти їх не обов'язково.
Вивід команд вставляти текстом у блоки коду, а не знімками екрана.
Порожні поля не допускаються: якщо щось не вдалося, записати, що саме і чому.
-->

# Лабораторна робота. DNS

| Поле | Значення |
|---|---|
| ПІБ | Томчак Дар'я Ігорівна|
| Група | ІПЗ-2.01 |
| Ідентифікатор (`sNN`) | se-201-31 |
| Дата виконання | 01.11.26 |
| Середовище | GitHub Codespaces |
| Версія `dig` | DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu |

## Параметри варіанта

<!-- Переписати з картки stand/cards/sNN.md. -->

| Параметр | Значення |
|---|---|
| `$TTL`, с | 75 |
| TTL запису SOA, с | 20 |
| Поле `minimum` SOA, с | 20 |
| Псевдонім (CNAME на `www`) | `shop` |
| Другий хост і його TTL, с | `canary`, 50 |
| Пріоритет MX | 10 |
| Токен TXT | `"lab-token=ef14f552"` |
| Ім'я для завдання 5 | `new-xjzrp.se-201-31.lab` |

## Завдання 1. Складові URI

### 1.1. Розбір

<!-- Відсутню складову позначати «—». Порт за замовчуванням записувати з позначкою «(за замовч.)». -->

| № | scheme | userinfo | host | port | path | query | fragment |
|---|---|---|---|---|---|---|---|
| 1 | http | - | www.se-201-31.lab | 80 (за замовч.) | /search/orders | limit=22&lang=uk | top |
| 2 | http | - | shop.se-201-31.lab | 80 | /orders/ | limit=%D0%BA%D0%B8%D1%97%D0%B2 | - |
| 3 | http | user:pass | canary.se-201-31.lab | 80 (за замовч.) | /search | - | comments |
| 4 | https | - | www.se-201-31.lab | 9443 | /search/../orders | - | - |
| 5 | mailto | - | - | - | admin@se-201-31.lab | subject=search | - |

| № | Ім'я, що запитується в DNS | Заголовок `Host` | Ціль запиту (request target) |
|---|---|---|---|
| 1 | www.se-201-31.lab | www.se-201-31.lab | /search/orders?limit=22&lang=uk |
| 2 | shop.se-201-31.lab | shop.se-201-31.lab | /orders/?limit=%D0%BA%D0%B8%D1%97%D0%B2 |
| 3 | canary.se-201-31.lab | canary.se-201-31.lab | /search |
| 4 | www.se-201-31.lab | www.se-201-31.lab:9443 | /search/../orders |
| 5 | - | - | - |

Декодоване значення параметра URI 2: limit=київ

Чому URI 5 не має складової host, хоча містить символ `@`: URI 5 використовує схему mailto, тому admin@se-201-31.lab є адресою електронної пошти. Символ @ тут є частиною email-адреси і не розділяє userinfo та host. Тому окремої складової host у цьому URI немає

### 1.2. Перевірка (після завдання 3)

<!-- Для URI 1–3: рядки запиту (починаються з ">") і тіло відповіді сервера. -->

```text
> GET /search/orders?limit=22&lang=uk HTTP/1.1
> Host: www.se-201-31.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Sat, 31 Oct 2026 13:55:30 GMT
< Content-Type: text/plain
< Content-Length: 146
< Connection: keep-alive
< 
server: web-a
Host: www.se-201-31.lab
request URI: /search/orders?limit=22&lang=uk
path: /search/orders
query: limit=22&lang=uk
client: 10.53.1.1
* Connection #0 to host www.se-201-31.lab left intact

> GET /orders/?limit=%D0%BA%D0%B8%D1%97%D0%B2 HTTP/1.1
> Host: shop.se-201-31.lab
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Sat, 31 Oct 2026 13:55:39 GMT
< Content-Type: text/plain
< Content-Length: 163
< Connection: keep-alive
< 
server: web-a
Host: shop.se-201-31.lab
request URI: /orders/?limit=%D0%BA%D0%B8%D1%97%D0%B2
path: /orders/
query: limit=%D0%BA%D0%B8%D1%97%D0%B2
client: 10.53.1.1
* Connection #0 to host shop.se-201-31.lab left intact

> GET /search HTTP/1.1
> Host: canary.se-201-31.lab
> Authorization: Basic dXNlcjpwYXNz
> User-Agent: curl/8.5.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Sat, 31 Oct 2026 13:55:51 GMT
< Content-Type: text/plain
< Content-Length: 102
< Connection: keep-alive
< 
server: web-b
Host: canary.se-201-31.lab
request URI: /search
path: /search
query: 
client: 10.53.1.1
* Connection #0 to host canary.se-201-31.lab left intact
```

Значення заголовка `Authorization` і результат декодування:

```text
> Authorization: Basic dXNlcjpwYXNz
root@ns-se-201-31:/# echo 'dXNlcjpwYXNz' | base64 -d      
user:passroot@ns-se-201-31:/# 
```

URI 4: етап, на якому запит не вдається (пошук імені / TCP / TLS), і рядки виводу, що це підтверджують:

```text
root@ns-se-201-31:/# curl -v 'https://www.se-201-31.lab:9443/search/../orders'
* Host www.se-201-31.lab:9443 was resolved.
* IPv6: (none)
* IPv4: 10.53.0.80
*   Trying 10.53.0.80:9443...
* connect to 10.53.0.80 port 9443 from 10.53.1.1 port 39836 failed: Connection refused
* Failed to connect to www.se-201-31.lab port 9443 after 0 ms: Couldn't connect to server
* Closing connection
curl: (7) Failed to connect to www.se-201-31.lab port 9443 after 0 ms: Couldn't connect to server
```

| Запит | `request URI`, отриманий сервером | `path`, отриманий сервером |
|---|---|---|
| звичайний | /orders | /orders |
| `--path-as-is` | /search/../orders | /orders |

Розбіжності між URI та тим, що отримав сервер, і їх пояснення: Для URI 1 і URI 3 частина після # не передається серверу. У URI 2 порт 80 не показується в Host, бо це стандартний порт для HTTP. У URI 3 user:pass передається в заголовку Authorization. Для URI 4 DNS спрацював, але з'єднання з портом 9443 не встановилося, тому TLS не почався. У звичайному запиті шлях /search/../orders змінився на /orders, а з --path-as-is сервер отримав повний шлях /search/../orders.

## Завдання 2. Проходження ієрархії вручну

```text
root@ns-se-201-31:/# dig @10.53.0.2 ns.se-201-31.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.2 ns.se-201-31.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59472
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 2cd82309ff8a26ec010000006adb535742198af3314e3b15 (good)
;; QUESTION SECTION:
;ns.se-201-31.lab.              IN      A

;; AUTHORITY SECTION:
lab.                    172800  IN      NS      ns1.nic.lab.

;; ADDITIONAL SECTION:
ns1.nic.lab.            172800  IN      A       10.53.0.3

;; Query time: 79 msec
;; SERVER: 10.53.0.2#53(10.53.0.2) (UDP)
;; WHEN: Fri Oct 23 12:30:15 UTC 2026
;; MSG SIZE  rcvd: 117

root@ns-se-201-31:/# dig @10.53.0.3 ns.se-201-31.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.3 ns.se-201-31.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 54080
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 9a2bd323896e7ae1010000006adb53f08776007aedd71c3f (good)
;; QUESTION SECTION:
;ns.se-201-31.lab.              IN      A

;; AUTHORITY SECTION:
se-201-31.lab.          3600    IN      NS      ns.se-201-31.lab.

;; ADDITIONAL SECTION:
ns.se-201-31.lab.       3600    IN      A       10.53.1.1

;; Query time: 79 msec
;; SERVER: 10.53.0.3#53(10.53.0.3) (UDP)
;; WHEN: Fri Oct 23 12:32:48 UTC 2026
;; MSG SIZE  rcvd: 103

root@ns-se-201-31:/# dig @10.53.1.1 ns.se-201-31.lab A +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 ns.se-201-31.lab A +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12140
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 119cfaa7212e56b4010000006adb53ff0e0fe134e52bb85e (good)
;; QUESTION SECTION:
;ns.se-201-31.lab.              IN      A

;; ANSWER SECTION:
ns.se-201-31.lab.       75      IN      A       10.53.1.1

;; AUTHORITY SECTION:
se-201-31.lab.          75      IN      NS      ns.se-201-31.lab.

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Fri Oct 23 12:33:03 UTC 2026
;; MSG SIZE  rcvd: 103

```

| Сервер | status | flags | ANSWER | AUTHORITY | ADDITIONAL |
|---|---|---|---|---|---|
| 10.53.0.2 | NOERROR | qr | 0 | lab.                    172800  IN      NS      ns1.nic.lab. | ns1.nic.lab.            172800  IN      A       10.53.0.3 <br> OPT |
| 10.53.0.3 | NOERROR | qr | 0 | se-201-31.lab.          3600    IN      NS      ns.se-201-31.lab. | ns.se-201-31.lab.       3600    IN      A       10.53.1.1 <br> OPT |
| 10.53.1.1 | NOERROR | qr aa | ns.se-201-31.lab.       75      IN      A       10.53.1.1 | se-201-31.lab.          75      IN      NS      ns.se-201-31.lab. | OPT |

Вивід `dig ns.sNN.lab A +trace`:

```text
root@ns-se-201-31:/# dig ns.se-201-31.lab A +trace

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> ns.se-201-31.lab A +trace
;; global options: +cmd
.                       85174   IN      NS      a.root-servers.net.
;; Received 59 bytes from 10.53.0.53#53(10.53.0.53) in 0 ms

lab.                    172800  IN      NS      ns1.nic.lab.
;; Received 117 bytes from 10.53.0.2#53(a.root-servers.net) in 80 ms

se-201-31.lab.          3600    IN      NS      ns.se-201-31.lab.
;; Received 103 bytes from 10.53.0.3#53(ns1.nic.lab) in 39 ms

ns.se-201-31.lab.       75      IN      A       10.53.1.1
se-201-31.lab.          75      IN      NS      ns.se-201-31.lab.
;; Received 103 bytes from 10.53.1.1#53(ns.se-201-31.lab) in 0 ms

```

| Запит | TTL запису NS | Прапорець `aa` | Секція, у якій міститься NS |
|---|---|---|---|
| `dig @10.53.0.3 sNN.lab NS +norec` | 3600 | нема | AUTHORITY |
| `dig @10.53.1.1 sNN.lab NS +norec` | 75 | є | ANSWER |

1. Які відповіді є відсиланнями і за якими ознаками це визначено?

> Відсиланнями є відповіді від 10.53.0.2 і 10.53.0.3. У них немає запису в ANSWER і немає прапорця aa, зате в AUTHORITY є запис NS, який вказує, до якого DNS сервера звертатися далі

2. Для чого у відповіді TLD присутній запис у секції ADDITIONAL і що сталося б, якби його не було?

> У ADDITIONAL є A-запис з IP-адресою DNS-сервера ns.se-201-31.lab, він потрібний, щоб резолвер одразу знав адресу наступного сервера, без нього довелося б окремо шукати IP-адресу цього NS

3. Яка з двох копій запису NS є авторитативною і чому?

> Авторитативною є копія від 10.53.1.1, тому що у відповіді є прапорець aa(Authoritative Answer)

## Завдання 3. Заповнення власної зони

Вивід `zone-reload`:

```text
root@ns-se-201-31:/# zone-reload
zone se-201-31.lab/IN: loaded serial 2026101902
OK
zone reload queued

```

<!-- Шість запитів до 10.53.1.1 з +norec, по одному на кожен рядок таблиці картки. -->

| Запис | status | Прапорець `aa` | Значення в ANSWER | TTL | Відповідає картці (так/ні) |
|---|---|---|---|---|---|
| `www` A | NOERROR | є | www.se-201-31.lab.      75      IN      A       10.53.0.80 | 75 | так |
| псевдонім CNAME | NOERROR | є | shop.se-201-31.lab.     75      IN      CNAME   www.se-201-31.lab. | 75 | так |
| `mail` A | NOERROR | є | mail.se-201-31.lab.     75      IN      A       10.53.0.80 | 75 | так |
| `@` MX | NOERROR | є | se-201-31.lab.          75      IN      MX      10 mail.se-201-31.lab. | 75 | так |
| `@` TXT | NOERROR | є | se-201-31.lab.          75      IN      TXT     "lab-token=ef14f552" | 75 | так |
| другий хост A | NOERROR | є | canary.se-201-31.lab.   50      IN      A       10.53.0.81 | 50 | так |

Порівняння відповідей резолвера та власного сервера:

```text
root@ns-se-201-31:/# dig shop.se-201-31.lab

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> shop.se-201-31.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19709
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;shop.se-201-31.lab.            IN      A

;; ANSWER SECTION:
shop.se-201-31.lab.     75      IN      CNAME   www.se-201-31.lab.
www.se-201-31.lab.      75      IN      A       10.53.0.80

;; Query time: 42 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Fri Oct 23 13:08:14 UTC 2026
;; MSG SIZE  rcvd: 81

root@ns-se-201-31:/# dig @10.53.1.1 shop.se-201-31.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 shop.se-201-31.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17439
;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: ce9ca4499768fea4010000006adb5c4c26cc6c453642a69f (good)
;; QUESTION SECTION:
;shop.se-201-31.lab.            IN      A

;; ANSWER SECTION:
shop.se-201-31.lab.     75      IN      CNAME   www.se-201-31.lab.
www.se-201-31.lab.      75      IN      A       10.53.0.80

;; AUTHORITY SECTION:
se-201-31.lab.          75      IN      NS      ns.se-201-31.lab.

;; ADDITIONAL SECTION:
ns.se-201-31.lab.       75      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Fri Oct 23 13:08:28 UTC 2026
;; MSG SIZE  rcvd: 142

root@ns-se-201-31:/# dig se-201-31.lab MX

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> se-201-31.lab MX
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46246
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;se-201-31.lab.                 IN      MX

;; ANSWER SECTION:
se-201-31.lab.          75      IN      MX      10 mail.se-201-31.lab.

;; Query time: 0 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Fri Oct 23 13:08:38 UTC 2026
;; MSG SIZE  rcvd: 63

root@ns-se-201-31:/# dig @10.53.1.1 se-201-31.lab MX +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 se-201-31.lab MX +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8171
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: cc33a818bfedc8b3010000006adb5c61078ee66c14ec9194 (good)
;; QUESTION SECTION:
;se-201-31.lab.                 IN      MX

;; ANSWER SECTION:
se-201-31.lab.          75      IN      MX      10 mail.se-201-31.lab.

;; AUTHORITY SECTION:
se-201-31.lab.          75      IN      NS      ns.se-201-31.lab.

;; ADDITIONAL SECTION:
mail.se-201-31.lab.     75      IN      A       10.53.0.80
ns.se-201-31.lab.       75      IN      A       10.53.1.1

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Fri Oct 23 13:08:49 UTC 2026
;; MSG SIZE  rcvd: 140

```

| Запит | flags | ANSWER (кількість, типи) | AUTHORITY (кількість, типи) | ADDITIONAL (кількість, типи) | TTL |
|---|---|---|---|---|---|
| псевдонім через resolver-a | qr rd ra | 2; CNAME, A | 0 | 1; OPT | 75 |
| псевдонім через 10.53.1.1 | qr aa | 2; CNAME, A | 1; NS | 2; A, OPT | 75 |
| MX через resolver-a | qr rd ra | 1; MX | 0 | 1; OPT | 75 |
| MX через 10.53.1.1 | qr aa | 1; MX | 1; NS |3; A, A, OPT | 75 |

Пояснення відмінностей: resolver-a має прапорці rd ra, тому що виконує рекурсивний пошук, але не має aa. Авторитативний сервер 10.53.1.1 має прапорець aa, додає NS у секцію AUTHORITY і може додавати A-записи в ADDITIONAL

Повідомлення `zone-check` після тимчасового додавання `@ IN CNAME www`:

```text
root@ns-se-201-31:/# zone-edit
root@ns-se-201-31:/# zone-check
dns_master_load: /lab/zone/se-201-31.lab.zone:21: se-201-31.lab: CNAME and other data
zone se-201-31.lab/IN: loading from master file /lab/zone/se-201-31.lab.zone failed: CNAME and other data
zone se-201-31.lab/IN: not loaded due to errors.
root@ns-se-201-31:/# zone-edit
root@ns-se-201-31:/# zone-check
zone se-201-31.lab/IN: loaded serial 2026101902
OK
root@ns-se-201-31:/# 
```

1. Який TTL має кожен запис зони і звідки це значення взято?

> Запис SOA має TTL 20 с, тому що він заданий явно як @ 20 IN SOA, записи NS, ns A, www A, shop CNAME, mail A, MX і TXT мають TTL 75 с, тому що для них не вказано окремий TTL і вони беруть значення 75 с за замовчуванням, запис canary A має TTL 50 с, тому що це значення окремо прописано для нього

2. Скільки записів містить секція ANSWER у відповіді на запит псевдоніма і чому?

> Два записи, CNAME для shop, який вказує на www, і A-запис www з адресою 10.53.0.80. це тому що запит до shop виконувався для типу A, тому у відповіді є і псевдонім, і кінцева IP-адреса

3. Чому запис CNAME неможливо розмістити для імені `@`?

> @ означає саме ім'я зони se-201-31.lab. Для нього вже існують інші записи, CNAME не може одночасно існувати з іншими даними для того самого імені, що підтверджує помилка CNAME and other data у zone-check

## Завдання 4. TTL і видимість змін

Два запити з інтервалом 5 с:

```text
root@ns-se-201-31:/# dig www.se-201-31.lab +noall +answer
www.se-201-31.lab.      75      IN      A       10.53.0.80
root@ns-se-201-31:/# dig www.se-201-31.lab +noall +answer
www.se-201-31.lab.      71      IN      A       10.53.0.80
```

**Прогноз** (записати до вимірювання). Формула часу появи нової адреси через залишок R, найбільша можлива затримка: очікуваний час = час зміни + R, найбільший можливий час затримки це 75, бо TTL = 75

Перевірка, що сервер до `zone-reload` повертає стару адресу:

```text
root@ns-se-201-31:/# dig @10.53.1.1 www.se-201-31.lab +norec +short
10.53.0.80
```

Залишок R і час (крок 4), час `zone-reload` (крок 5), перевірка на сервері (крок 6):

```text
root@ns-se-201-31:/# date +%T; dig www.se-201-31.lab +noall +answer
15:14:45
www.se-201-31.lab.      25      IN      A       10.53.0.80
root@ns-se-201-31:/# date +%T; dig www.se-201-31.lab +noall +answer
15:15:20
www.se-201-31.lab.      75      IN      A       10.53.0.80
root@ns-se-201-31:/# zone-reload
zone se-201-31.lab/IN: loaded serial 2026101903
OK
zone reload queued
root@ns-se-201-31:/# date +%T
15:15:32
root@ns-se-201-31:/# dig @10.53.1.1 www.se-201-31.lab +norec +short
10.53.0.81
```

Очікуваний час появи нової адреси (підставити R у формулу): 15:15:32 + 75 = 15:16:47

Вивід циклу спостереження (перші два рядки, два останні рядки зі старою адресою, рядок з новою адресою):

```text
root@ns-se-201-31:/# while true; do echo "$(date +%T) $(dig +short www.se-201-31.lab)"; sleep 2; done
15:15:46 10.53.0.80
15:15:48 10.53.0.80

15:16:32 10.53.0.80
15:16:34 10.53.0.80
15:16:36 10.53.0.81
^C
```

Рядок `server:` з відповіді `curl`: server: web-b

| Час зміни | Очікуваний час | Спостережений час | Виміряна затримка, с | Прогноз, с |
|---|---|---|---|---|
| 15:15:32 | 15:16:47 | 15:16:36 | 64 | 75 |

1. Чим пояснюється розбіжність між прогнозом і виміряним значенням?

> Між вимірюванням TTL і виконанням zone-reload пройшло кілька секунд, тому TTL за цей час уже зменшився. Також перевірка виконувалась раз на 2 секунди, тому точний момент зміни міг бути пропущений

2. Якою була б найбільша можлива затримка для цього запису і за яких умов?

> Найбільша затримка 75 секунд, вона була б тоді, коли б запис щойно потрапив у кеш і його TTL дорівнював би повним 75 секундам

3. Як слід діяти, щоб перенести сервіс на нову адресу, якщо TTL запису становить 86 400 с?

> Спочатку потрібно зменшити TTL, зачекати, поки старий TTL закінчиться в кешах, і тільки після цього змінювати адресу, так клієнти швидше отримають нову адресу

## Завдання 5. Негативне кешування

**Прогноз** (записати до вимірювання). Формула за RFC 2308 і розраховане значення: негативне кешування = min(SOA TTL, SOA minimum) = (20, 20) = 20 c

Перевірка, що сервер до `zone-reload` повертає `NXDOMAIN`:

```text
root@ns-se-201-31:/# dig @10.53.1.1 new-xjzrp.se-201-31.lab +norec

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 new-xjzrp.se-201-31.lab +norec
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 22762
;; flags: qr aa; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: b91b0b2d8b703e37010000006ae627ce77812b528686cb29 (good)
;; QUESTION SECTION:
;new-xjzrp.se-201-31.lab.       IN      A

;; AUTHORITY SECTION:
se-201-31.lab.          20      IN      SOA     ns.se-201-31.lab. admin.se-201-31.lab. 2026101903 3600 600 86400 20

;; Query time: 0 msec
;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)
;; WHEN: Sat Oct 31 17:39:26 UTC 2026
;; MSG SIZE  rcvd: 125
```

Запит через resolver-a (крок 3), час `zone-reload` (крок 4), перевірка на сервері (крок 5):

```text
root@ns-se-201-31:/# date +%T; dig new-xjzrp.se-201-31.lab
17:39:28

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> new-xjzrp.se-201-31.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 14434
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;new-xjzrp.se-201-31.lab.       IN      A

;; AUTHORITY SECTION:
se-201-31.lab.          20      IN      SOA     ns.se-201-31.lab. admin.se-201-31.lab. 2026101903 3600 600 86400 20

;; Query time: 0 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Sat Oct 31 17:39:28 UTC 2026
;; MSG SIZE  rcvd: 97

root@ns-se-201-31:/# zone-reload; date +%T
zone se-201-31.lab/IN: loaded serial 2026101904
OK
zone reload queued
17:39:33
root@ns-se-201-31:/# dig @10.53.1.1 new-xjzrp.se-201-31.lab +norec +short
10.53.0.80
```

Вивід циклу спостереження (перші два рядки, останній рядок з `NXDOMAIN`, перший рядок з `NOERROR`):

```text
root@ns-se-201-31:/# while true; do echo "$(date +%T) $(dig new-xjzrp.se-201-31.lab +noall +comments | grep -o 'status: [A-Z]*')"; sleep 2; done
17:39:43 status: NXDOMAIN
17:39:45 status: NXDOMAIN
17:39:47 status: NXDOMAIN
17:39:49 status: NOERROR
^C
```

| Негативну відповідь закешовано | Запис створено | Ім'я резолвиться через resolver-a | Виміряно, с | Прогноз, с |
|---|---|---|---|---|
| 17:39:28 | 17:39:33 | 17:39:49 | 16 | 20 |

<!-- Якщо завдання повторювалося з іншим ім'ям (new2-…), зазначити це й причину. -->

1. Чому тривалість негативного кешування визначається записом SOA, а не записом, який запитували?

> Тому що запитуваного запису ще не існує і він не має власного TTL, а для відповіді NXDOMAIN резолвер використовує інформацію із SOA зони, негативний TTL визначається як min(20, 20) = 20 с

2. Чи збігається TTL запису SOA в секції AUTHORITY з TTL запису SOA у файлі зони? Поясніть.

> Так, у TTL SOA дорівнює 20 с, і у відповіді NXDOMAIN у секції AUTHORITY також було 20 IN SOA, у моєму варіанті TTL SOA і поле minimum однакові

3. Скрипт розгортання перевіряє ім'я сервісу до створення запису, а потім одразу після. Що він побачить і як це виправити?

> До створення запису скрипт отримає NXDOMAIN, ця відповідь може потрапити в кеш, тому одразу після створення запису він ще деякий час також може бачити NXDOMAIN, потрібно дочекатися спливання негативного TTL або робити повторні перевірки

## Завдання 6. Затримка пошуку імені

**Прогноз** (записати до вимірювання). До яких серверів звернеться resolver-a:

| Ім'я | Перший запит | Повторний запит |
|---|---|---|
| A: нове ім'я у власній зоні | 10.53.1.1 | кеш resolver-a |
| B: нове ім'я в зоні `lab.` | 10.53.0.3 | кеш resolver-a |
| C: ім'я в неіснуючому TLD | 10.53.0.2 | кеш resolver-a |
| D: `www.sNN.lab` після перезапуску резолвера | спочатку 10.53.0.2 потім 10.53.0.3 потім 10.53.1.1 | кеш resolver-a |

Вивід вимірювань:

```text
root@ns-se-201-31:/# A=y$RANDOM.se-201-31.lab; dig $A | grep 'Query time'; dig $A | grep 'Query time'
;; Query time: 1 msec
;; Query time: 0 msec

root@ns-se-201-31:/# B=x$RANDOM.lab; dig $B | grep 'Query time'; dig $B | grep 'Query time'
;; Query time: 40 msec
;; Query time: 0 msec

root@ns-se-201-31:/# C=x$RANDOM.zz$RANDOM; dig $C | grep 'Query time'; dig $C | grep 'Query time'
;; Query time: 80 msec
;; Query time: 0 msec

root@ns-se-201-31:/# dig www.se-201-31.lab | grep 'Query time'
;; Query time: 404 msec
root@ns-se-201-31:/# dig www.se-201-31.lab | grep 'Query time'
;; Query time: 0 msec
```

| Ім'я | Перший запит, мс | Повторний, мс | Сервери, до яких звернувся резолвер (за журналами) |
|---|---|---|---|
| A | 1 | 0 | 10.53.1.1 |
| B | 40 | 0 | 10.53.0.3 |
| C | 80 | 0 | 10.53.0.2 |
| D | 404 | 0 | спочатку 10.53.0.2 потім 10.53.0.3 потім 10.53.1.1 |

Значення `time_namelookup` з `curl` після перезапуску резолвера: 0.405717

Фрагменти журналів запитів root, TLD і власного сервера для рядка D:

```text
root:
01-Nov-2026 11:05:49.827 client @0x754e6806ac88 10.53.0.53#42973 (.): query: . IN NS -E(0)DC (10.53.0.2)
01-Nov-2026 11:05:49.907 client @0x754e60006078 10.53.0.53#61852 (a.root-servers.net): query: a.root-servers.net IN A -E(0)DC (10.53.0.2)
01-Nov-2026 11:05:49.988 client @0x754e6806ac88 10.53.0.53#41512 (www.se-201-31.lab): query: www.se-201-31.lab IN A -E(0)DC (10.53.0.2)

TLD:
01-Nov-2026 11:05:50.109 client @0x7ee31406ac88 10.53.0.53#42904 (www.se-201-31.lab): query: www.se-201-31.lab IN A -E(0)DC (10.53.0.3)

власний авторитативний сервер:
01-Nov-2026 11:05:50.149 client @0x7a9fec06ae18 10.53.0.53#32346 (www.se-201-31.lab): query: www.se-201-31.lab IN A -E(0)DC (10.53.1.1)
```

1. Яку затримку вносить кожен із серверів root, TLD і власний сервер? Покажіть, з яких рядків таблиці це випливає.

> Власний авторитативний сервер дає приблизно 1 мс це видно з рядка A, TLD дає приблизно 40 мс це рядок B, root дає приблизно 80 мс це рядок C

2. Чому в рядку A перший запит не проходить через root і TLD, хоча ім'я нове?

> Тому що resolver-a уже знає, де знаходиться зона se-201-31.lab, новим є лише саме ім'я хоста, тому резолвер може одразу звернутися до авторитативного сервера, не проходячи root і TLD

3. Вебсторінка завантажує ресурси з п'яти різних доменів. Як це впливає на час першого відкриття сторінки?

> Для кожного нового домену може знадобитися окремий DNS-пошук, через що перше відкриття сторінки може бути довшим, а після того як адреси потраплять у кеш наступне відкриття буде швидшим

## Завдання 7. Інцидент: зупинка власного сервера

**Прогноз** (записати до вимірювання). Що поверне кожен резолвер після зупинки сервера та спливання TTL і скільки часу займе відповідь: Після спливання TTL resolver-a не зможе отримати свіжу відповідь, буде тривале очікування аж до заданого timeout=30 с, після чого можливий тайм-аут або SERVFAIL, resolver-b завдяки serve-stale має повернути старий запис значно швидше за 30 с, орієнтовно за кілька секунд

Кешування запису в обох резолверах (крок 2) і час зупинки сервера (крок 3):

```text
root@ns-se-201-31:/# dig @10.53.0.53 canary.se-201-31.lab +noall +answer
canary.se-201-31.lab.   50      IN      A       10.53.0.81
root@ns-se-201-31:/# dig @10.53.0.54 canary.se-201-31.lab +noall +answer
canary.se-201-31.lab.   50      IN      A       10.53.0.81
root@ns-se-201-31:/# named-stop
root@ns-se-201-31:/# date +%T
19:46:22
```

Запити із заміром часу (крок 5):

```text
root@ns-se-201-31:/# time dig @10.53.0.53 canary.se-201-31.lab +tries=1 +timeout=30
;; communications error to 10.53.0.53#53: timed out

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.53 canary.se-201-31.lab +tries=1 +timeout=30
; (1 server found)
;; global options: +cmd
;; no servers could be reached

real    0m30.045s
user    0m0.006s
sys     0m0.004s
root@ns-se-201-31:/# time dig @10.53.0.53 canary.se-201-31.lab +tries=1 +timeout=30

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.53 canary.se-201-31.lab +tries=1 +timeout=30
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 27725
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;canary.se-201-31.lab.          IN      A

;; Query time: 0 msec
;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)
;; WHEN: Sat Oct 31 19:47:58 UTC 2026
;; MSG SIZE  rcvd: 49


real    0m0.021s
user    0m0.006s
sys     0m0.005s
root@ns-se-201-31:/# time dig @10.53.0.54 canary.se-201-31.lab +tries=1 +timeout=30

; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.0.54 canary.se-201-31.lab +tries=1 +timeout=30
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45653
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;canary.se-201-31.lab.          IN      A

;; ANSWER SECTION:
canary.se-201-31.lab.   30      IN      A       10.53.0.81

;; Query time: 1800 msec
;; SERVER: 10.53.0.54#53(10.53.0.54) (UDP)
;; WHEN: Sat Oct 31 19:48:05 UTC 2026
;; MSG SIZE  rcvd: 65


real    0m1.822s
user    0m0.006s
sys     0m0.005s
```

| Запит | status або тайм-аут | ANSWER, TTL | `real` |
|---|---|---|---|
| resolver-a, перший | timeout | немає | 30.045 |
| resolver-a, другий | SERVFAIL | немає | 0.021 |
| resolver-b | NOERROR | canary.se-201-31.lab.   30      IN      A       10.53.0.81 | 1.822 |

Час `named-start` і вивід циклу спостереження до першої успішної відповіді (крок 6):

```text
root@ns-se-201-31:/# named-start
root@ns-se-201-31:/# date +%T
19:48:12
root@ns-se-201-31:/# while true; do echo "$(date +%T) $(dig @10.53.0.53 canary.se-201-31.lab +short +tries=1 +timeout=3)"; sleep 2; done
19:48:16 
19:48:18 
19:48:20 
19:48:22 
19:48:24 
19:48:27 
19:48:29 10.53.0.81
^C
```

Перевірка після завдання (`dig @10.53.1.1 sNN.lab SOA +norec +short`):

```text
root@ns-se-201-31:/# dig @10.53.1.1 se-201-31.lab SOA +norec +short
ns.se-201-31.lab. admin.se-201-31.lab. 2026101904 3600 600 86400 20
```

### Постмортем

1. Що сталося: авторитативний сервер 10.53.1.1 був зупинений командою named-stop, тому після спливання TTL резолвери не могли отримати свіжий запис canary
2. Вплив на клієнтів resolver-a і resolver-b: resolver-a спочатку дав тайм-аут, потім SERVFAIL, а resolver-b завдяки serve-stale повернув стару адресу 10.53.0.81
3. Хронологія з виміряним часом: сервер зупинено о 19:46:22, перший запит resolver-a тривав 30.045 с, другий запит тривав 0.021 с, resolver-b відповів за 1.822 с, сервер запущено о 19:48:12, перша успішна відповідь resolver-a з'явилася о 19:48:29
4. Чому кеш допоміг або не допоміг: звичайний resolver-a не використовував прострочений запис, тому після спливання TTL клієнт не отримав адресу, а resolver-b зберіг прострочений запис і зміг тимчасово віддати його клієнту
5. Що слід змінити: Для більшої надійності краще мати резервний авторитативний DNS-сервер
---

1. Який TTL мав запис у відповіді resolver-b і навіщо такий TTL?

> TTL 30 с, такий невеликий TTL потрібен, щоб клієнт не зберігав стару адресу надовго і скоро знову отримав актуальну відповідь

2. Порівняйте результат і тривалість двох запитів до resolver-a. Якщо вони різняться, сформулюйте гіпотезу щодо причини; якщо ні, поясніть, що робить резолвер протягом цього часу.

> Перший запит завершився тайм-аутом і тривав 30.045 с, а другий запит майже одразу повернув SERVFAIL за 0.021 с, ймовірно, після першої невдалої спроби resolver-a вже знав, що авторитативний сервер недоступний, тому не чекав 30 секунд

3. Чому після `named-start` ім'я через resolver-a зарезолвилося не одразу?

> Після запуску сервера ймовірно resolver-a ще деякий час пам'ятав попередню невдалу спробу, тому він не одразу звернувся до сервера повторно. Сервер запустили о 19:48:12, а адреса знову з'явилася о 19:48:29, тобто приблизно через 17 с

4. У якій ситуації serve-stale шкодить, а не допомагає? Пов'яжіть відповідь із завданням 4.

> Serve-stale може шкодити, якщо застарілий запис уже веде на неправильну адресу. У завданні 4 www було перенесено з 10.53.0.80 на 10.53.0.81, якщо резолвер продовжить віддавати stale-запис зі старою адресою, клієнти можуть потрапляти на старий або вже вимкнений сервер

## Фінальний файл зони

<!-- Повний вміст stand/students/sNN/zone/sNN.lab.zone після завершення роботи. -->

```text
root@ns-se-201-31:/# cat /lab/zone/se-201-31.lab.zone
; Zone se-201-31.lab — starting version.
; After every change: increase the serial, then run  zone-reload
$TTL 75
@     20 IN SOA ns.se-201-31.lab. admin.se-201-31.lab. (
            2026101904 ; serial
            3600       ; refresh
            600        ; retry
            86400      ; expire
            20         ; minimum
        )
@     IN NS ns.se-201-31.lab.
ns    IN A  10.53.1.1

; ---- add records from your variant card below this line ----
www                 IN A      10.53.0.81
shop                IN CNAME  www
mail                IN A      10.53.0.80
@                   IN MX     10 mail
@                   IN TXT    "lab-token=ef14f552"
canary      50       IN A      10.53.0.81
new-xjzrp    IN A    10.53.0.80
```

## Висновки

**Що виявилося несподіваним.** Результат, який не збігся з очікуванням, і рядок виводу, де це видно: У завданні 7 два однакові запити до resolver-a після зупинки авторитативного сервера дали дуже різний результат: перший завершився тайм-аутом за 30.045 с, а другий майже одразу повернув SERVFAIL за 0.021 с. Тобто виходить, що поведінка повторного запиту може змінитися після вже виконаної невдалої рекурсії

**Яке питання залишилося без відповіді.** Що саме не вдалося пояснити; до якого рядка виводу це стосується; що вже перевірено (RFC, документація, додатковий експеримент на стенді) і що з'ясувати не вдалося: Чому resolver-a після першого довгого тайм-ауту при повторному запиті майже одразу повернув SERVFAIL і скільки часу він зберігає інформацію про таку невдалу спробу?

## Декларація використання ШІ

Використовувалися інструменти ШІ: так

<!-- Якщо так, заповнити таблицю: один рядок на кожне звернення, що вплинуло на звіт. Промпт наводити дослівно. -->

| № | Інструмент | Промпт (дослівно) | Що використано у звіті і що змінено самостійно |
|---|---|---|---|
| 1 | ChatGPT Plus | проаналізуй роботу, яку мені потрібно буде виконати, детально розпиши теоретичну частину, що допоможе мені з виконанням та додай поради, щоб уникнути помилок(у цьому запиті було додано два файли: протокол для виконання лабораторної роботи та методичні вказівки) | Надану теорію було проаналізовано та використано в подальшому виконанні завдань, також було відхилено пропозицію щодо допомоги зі заповненням звіту |
| 2 | ChatGPT Plus | чому під час заповнення таблиці виявляється, що в полі ADDITIONAL не вистачає ще одного типу? тобто зазначається кількість на 1 більша, ніж є в секції, від чого це залежить, як це можна пояснити та яким є той самий тип?:<br><br>root@ns-se-201-31:/# dig shop.se-201-31.lab<br><br>; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> shop.se-201-31.lab<br>;; global options: +cmd<br>;; Got answer:<br>;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19709<br>;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1<br><br>;; OPT PSEUDOSECTION:<br>; EDNS: version: 0, flags:; udp: 1232<br>;; QUESTION SECTION:<br>;shop.se-201-31.lab. IN A<br><br>;; ANSWER SECTION:<br>shop.se-201-31.lab. 75 IN CNAME www.se-201-31.lab.<br>www.se-201-31.lab. 75 IN A 10.53.0.80<br><br>;; Query time: 42 msec<br>;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)<br>;; WHEN: Fri Oct 23 13:08:14 UTC 2026<br>;; MSG SIZE rcvd: 81<br><br>root@ns-se-201-31:/# dig @10.53.1.1 shop.se-201-31.lab +norec<br><br>; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 shop.se-201-31.lab +norec<br>; (1 server found)<br>;; global options: +cmd<br>;; Got answer:<br>;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17439<br>;; flags: qr aa; QUERY: 1, ANSWER: 2, AUTHORITY: 1, ADDITIONAL: 2<br><br>;; OPT PSEUDOSECTION:<br>; EDNS: version: 0, flags:; udp: 1232<br>; COOKIE: ce9ca4499768fea4010000006adb5c4c26cc6c453642a69f (good)<br>;; QUESTION SECTION:<br>;shop.se-201-31.lab. IN A<br><br>;; ANSWER SECTION:<br>shop.se-201-31.lab. 75 IN CNAME www.se-201-31.lab.<br>www.se-201-31.lab. 75 IN A 10.53.0.80<br><br>;; AUTHORITY SECTION:<br>se-201-31.lab. 75 IN NS ns.se-201-31.lab.<br><br>;; ADDITIONAL SECTION:<br>ns.se-201-31.lab. 75 IN A 10.53.1.1<br><br>;; Query time: 0 msec<br>;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)<br>;; WHEN: Fri Oct 23 13:08:28 UTC 2026<br>;; MSG SIZE rcvd: 142<br><br>root@ns-se-201-31:/# dig se-201-31.lab MX<br><br>; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> se-201-31.lab MX<br>;; global options: +cmd<br>;; Got answer:<br>;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 46246<br>;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1<br><br>;; OPT PSEUDOSECTION:<br>; EDNS: version: 0, flags:; udp: 1232<br>;; QUESTION SECTION:<br>;se-201-31.lab. IN MX<br><br>;; ANSWER SECTION:<br>se-201-31.lab. 75 IN MX 10 mail.se-201-31.lab.<br><br>;; Query time: 0 msec<br>;; SERVER: 10.53.0.53#53(10.53.0.53) (UDP)<br>;; WHEN: Fri Oct 23 13:08:38 UTC 2026<br>;; MSG SIZE rcvd: 63<br><br>root@ns-se-201-31:/# dig @10.53.1.1 se-201-31.lab MX +norec<br><br>; <<>> DiG 9.18.39-0ubuntu0.24.04.8-Ubuntu <<>> @10.53.1.1 se-201-31.lab MX +norec<br>; (1 server found)<br>;; global options: +cmd<br>;; Got answer:<br>;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8171<br>;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 3<br><br>;; OPT PSEUDOSECTION:<br>; EDNS: version: 0, flags:; udp: 1232<br>; COOKIE: cc33a818bfedc8b3010000006adb5c61078ee66c14ec9194 (good)<br>;; QUESTION SECTION:<br>;se-201-31.lab. IN MX<br><br>;; ANSWER SECTION:<br>se-201-31.lab. 75 IN MX 10 mail.se-201-31.lab.<br><br>;; AUTHORITY SECTION:<br>se-201-31.lab. 75 IN NS ns.se-201-31.lab.<br><br>;; ADDITIONAL SECTION:<br>mail.se-201-31.lab. 75 IN A 10.53.0.80<br>ns.se-201-31.lab. 75 IN A 10.53.1.1<br><br>;; Query time: 0 msec<br>;; SERVER: 10.53.1.1#53(10.53.1.1) (UDP)<br>;; WHEN: Fri Oct 23 13:08:49 UTC 2026<br>;; MSG SIZE rcvd: 140<br><br>\|Запит \| flags \| ANSWER (кількість, типи) \| AUTHORITY (кількість, типи) \| ADDITIONAL (кількість, типи) \| TTL\|<br>\|псевдонім через resolver-a \| qr rd ra \| 2; CNAME, A \| 0 \| 1; \| 75\|<br>\|псевдонім через 10.53.1.1 \| qr aa \| 2; CNAME, A \| 1; NS \| 2; A \| 75\|<br>\|MX через resolver-a \| qr rd ra \| 1; MX \| 0 \| 1; \| 75\|<br>\|MX через 10.53.1.1 \| qr aa \| 1; MX \| 1; NS \| 3; A, A \| 75 \| | Ші зазначив, що у моєму випадку це OPT - службовий запис EDNS, який технічно рахується до ADDITIONAL, але знаходиться в OPT PSEUDOSECTION. Я використала цю інформацію та додала до таблиці OPT |
