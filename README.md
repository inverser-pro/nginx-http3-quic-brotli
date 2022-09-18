# Установка и настройка nginx + quic + brotli (openssl+quic + boringssl)

### [Моя видео инструкция](https://www.youtube.com/watch?v=K_p2nJbQVdc "Тестирование TLS соединения")

###  Результат
[![](https://raw.githubusercontent.com/inverser-pro/nginx-http3-quic-brotli/main/images/2022-09-18_02-48.png)](https://raw.githubusercontent.com/inverser-pro/nginx-http3-quic-brotli/main/images/2022-09-18_02-48.png)

**Внимание, друзья!**

В папке ```nginx_conf``` лежат два файла настроек для nginx! Проведите изменения в них таким образом: укажите, где находится Ваш root (папка с сайтом > nginx.conf), укажите ВАШЕ название домена; и также в файле ```http3.conf``` поменяйте пути к сертификату TLS (Let's encrypt)

**Система** ```lsb_release -da``` :
- Debian 11

[Здесь написано отлично, как можно обновить систему и все пакеты](https://linuxize.com/post/how-to-upgrade-debian-10-to-debian-11/)

**Ядро** ```uname -srm```:
- Linux 5.10.0-18-amd64 x86_64

Это всё пригодилось для результата...

https://github.com/VKCOM/nginx-quic

[Конфигуратор алгоритмов шифрования от Mozilla](https://ssl-config.mozilla.org "Конфигуратор алгоритмов шифрования от Mozilla")

[Тестирование TLS соединения](https://www.ssllabs.com/ssltest/analyze.html "Тестирование TLS соединения")

https://risaksson.com/post/9/2022-05-10/HTTP3-with-NGINX

https://www.linuxcapable.com/how-to-install-latest-nginx-mainline-or-stable-on-debian-11/

https://github.com/VKCOM/nginx-quic

https://www.nginx.com/blog/our-roadmap-quic-http-3-support-nginx/

https://github.com/quictls/openssl

https://github.com/google/ngx_brotli

Для конфигурации brotli
https://github.com/pepelsbey/playground/blob/main/56/1-server.md

nginx настройка
https://www.vultr.com/docs/nginx-redirects-for-non-www-sub-domains-to-www/

https://driesdeboosere.dev/blog/how-to-redirect-www-to-non-www-with-nginx/

Почитать:

https://github.com/ngtcp2/ngtcp2

https://www.grottedubarbu.fr/nginx-quic-http3/

https://cabulous.medium.com/http-3-quic-and-how-it-works-c5ffdb6735b4

https://www.smashingmagazine.com/2021/08/http3-core-concepts-part1/

boringssl установка (отдельно)
https://gist.github.com/neilstuartcraig/4b8f06a4d4374c379bc0f44923a11fa4

----------

Логика действий:

- Скачиваем и устанавливаем openssl+quic текущей версии.
- Скачиваем модуль nginx + brotli
- Скачиваем исходники nginx с флагом -b quic
- Скачиваем boringssl — это как бы для quic
- Устанавливаем все необходимые зависимости для установки boringssl, nginx, openssl
- Устанавливаем сначала boringssl
- Затем устанавливаем nginx с конфигурацией, как у меня, с указанием путей к исходникам openssl и модулю ngx_brotli

- Устанавливаем certbot
- Настраиваем конфигурацию сервера, тестируем её, запускаем сервер
- Проверяем http2, http3, brotli

Можно также поставить sshguard для улучшения безопасности сервера

-----

Авторизируемся в своей программе через SSH на своём сервере...
Обычно сразу кидает в папку /root

Вот в ней (/root) выполняем следующие команды...

### Модуль nginx ngx_brotli
```
git clone https://github.com/google/ngx_brotli.git
```
*его просто необходимо скопировать*

## Скачиваем и устанавливаем openssl+quic
```
git clone --depth 1 -b openssl-3.0.5+quic https://github.com/quictls/openssl
cd openssl && \
./config enable-tls1_3 --prefix=$PWD/build
make -j$(nproc)
make install_sw
```
#### Выполняем для установки boringssl
```
 apt-get update && \
    apt-get install -y git gcc make g++ cmake perl libunwind-dev golang && \
    git clone https://boringssl.googlesource.com/boringssl && \
    mkdir boringssl/build && \
    cd boringssl/build && \
    cmake .. && \
    make
```
#### Установим сам nginx + модуль brotli + openssl+quic
```
 apt-get install -y mercurial libperl-dev libpcre3-dev zlib1g-dev libxslt1-dev libgd-ocaml-dev libgeoip-dev && \
    hg clone -b quic https://hg.nginx.org/nginx-quic   && \
    hg clone http://hg.nginx.org/njs -r "0.6.2" && \
    cd nginx-quic && \
    hg update quic && \
    auto/configure `nginx -V 2>&1 | sed "s/ \-\-/ \\\ \n\t--/g" | grep "\-\-" | grep -ve opt= -e param= -e build=` \
        --prefix=/etc/nginx \
        --sbin-path=/usr/sbin/nginx \
        --with-openssl=../openssl \
        --conf-path=/etc/nginx/nginx.conf \
        --http-log-path=/var/log/nginx/access.log \
        --error-log-path=/var/log/nginx/error.log \
        --with-pcre  \
        --lock-path=/var/lock/nginx.lock \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module \
        --with-http_image_filter_module=dynamic \
        --modules-path=/etc/nginx/modules \
        --with-http_v2_module \
        --with-stream=dynamic \
        --with-http_addition_module \
        --with-http_mp4_module  \
        --add-module=../ngx_brotli \
        --build=nginx-quic --with-debug  \
        --with-http_v3_module --with-stream_quic_module \
        --with-cc-opt="-I/src/boringssl/include" --with-ld-opt="-L/src/boringssl/build/ssl -L/src/boringssl/build/crypto" && \
    make && make install
```

Проверяем Brotli

```curl -H 'Accept-Encoding: br' -I http://localhost```

https://github.com/google/ngx_brotli

https://ruvds.com/ru/helpcenter/ubuntu-debian-packages/

https://linuxize.com/post/how-to-upgrade-debian-10-to-debian-11/

На всякий
```
apt install mercurial libpcre3 libpcre3-dev gcc make autoconf zlib1g zlib1g-dev -y
```
Для модуля nginx brotli (на всякий)
```
cd ../ngx_brotli && git submodule update --init && cd /root/nginx-quic
```
## Видосики

Николай (Метод Лаб) — полная настройка http3

https://www.youtube.com/watch?v=XwdCL_YjJMo

Вадим Макеев — полная настройка https http2 brotli + безопасного сервера

https://www.youtube.com/watch?v=XDWWxA6g1eA

Anton Putra установка certbot (НАДО)

https://www.youtube.com/watch?v=R5d-hN9UtpU

## Установка certbot

Выполняем каждую команду отдельно
```
apt update
apt policy snapd
apt install snapd
snap install core; snap update core
snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
certbot --version
certbot --nginx
    enter_your_valid@email
    y, y, y
    1 2
```
Для конфигурирования nginx (выше есть, но на всякий =)) ):
```
auto/configure `nginx -V 2>&1 | sed "s/ \-\-/ \\\ \n\t--/g" | grep "\-\-" | grep -ve opt= -e param= -e build=` \
    --prefix=/etc/nginx \
    --sbin-path=/usr/sbin/nginx \
    --with-openssl=../openssl \
    --conf-path=/etc/nginx/nginx.conf \
    --http-log-path=/var/log/nginx/access.log \
    --error-log-path=/var/log/nginx/error.log \
    --with-pcre  \
    --lock-path=/var/lock/nginx.lock \
    --pid-path=/var/run/nginx.pid \
    --with-http_ssl_module \
    --with-http_image_filter_module=dynamic \
    --modules-path=/etc/nginx/modules \
    --with-http_v2_module \
    --with-stream=dynamic \
    --with-http_addition_module \
    --with-http_mp4_module  \
    --add-module=../ngx_brotli \
    --build=nginx-quic --with-debug  \
    --with-http_v3_module --with-stream_quic_module \
    --with-cc-opt="-I/src/boringssl/include" --with-ld-opt="-L/src/boringssl/build/ssl -L/src/boringssl/build/crypto"
```
### Генерируем dhparam
```openssl dhparam -out /etc/nginx/crt/dhparam.pem 4096```

## Такая ошибка возникала из-за малого объема оперативы, при установке boringssl:

(bugs.chromium.org/p/boringssl/issues/detail?id=520)

```
[ 86%] Linking CXX static library libssl.a
[ 86%] Built target ssl
Scanning dependencies of target ssl_test
[ 86%] Building CXX object ssl/CMakeFiles/ssl_test.dir/span_test.cc.o
[ 86%] Building CXX object ssl/CMakeFiles/ssl_test.dir/ssl_test.cc.o
c++: fatal error: Killed signal terminated program cc1plus
compilation terminated.
make[2]: *** [ssl/CMakeFiles/ssl_test.dir/build.make:76: ssl/CMakeFiles/ssl_test.dir/ssl_test.cc.o] Error 1
make[1]: *** [CMakeFiles/Makefile2:602: ssl/CMakeFiles/ssl_test.dir/all] Error 2
make: *** [Makefile:130: all] Error 2
root@devq:~/boringssl/build# yum -y install gcc-c++
-bash: yum: command not found
root@devq:~/boringssl/build# apt install yum
Reading package lists... Done
Building dependency tree
Reading state information... Done
```

На всякий:

```
apt update && apt upgrade -y
apt install wget gnupg2 ca-certificates lsb-release debian-keyring software-properties-common -y
```
```
wget -O- https://nginx.org/keys/nginx_signing.key | gpg --dearmor | tee /usr/share/keyrings/nginx-archive-keyring.gpg
```
```
echo deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/debian `lsb_release -cs` nginx | tee /etc/apt/sources.list.d/nginx-mainline.list
```
```
echo deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian `lsb_release -cs` nginx | tee /etc/apt/sources.list.d/nginx-stable.list
```
```
echo deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/debian `lsb_release -cs` nginx | tee /etc/apt/sources.list.d/nginx-stable.list
```
```
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" | tee /etc/apt/preferences.d/99nginx
```
```
lfache@Midgar:~/nginx-quic$ cd ..
lfache@Midgar:~$ git clone --depth 1 -b OpenSSL_1_1_1g-quic-v1-compat https://github.com/tatsuhiro-t/openssl
lfache@Midgar:~$ cd openssl
lfache@Midgar:~/openssl$ ./config enable-tls1_3 --openssldir=/etc/ssl
lfache@Midgar:~/openssl$ make -j$(nproc)
lfache@Midgar:~/openssl$ sudo make install_sw
```
Была такая ошибка. Пришлось переустановить opessl на openssl+quic (v3.0.5)
```nginx: /lib/x86_64-linux-gnu/libssl.so.1.1: version `OPENSSL_1_1_1d' not found (required by nginx)```

Чтобы сохранить список установленных программ:
```
dpkg --get-selections | grep -v deinstall > mylist.txt
```
-----

### доп. инфо

#### SSHGuard

https://www.youtube.com/watch?v=-GTkM6XFEBw

Команды из видео по установке SSHGuard
```
echo 'deb http://deb.debian.org/debian sid main' > \
/etc/apt/sources.list.d/sid.list
```
```
echo 'Package: *
Pin: release a=unstable
Pin-Priority: -100' > /etc/apt/preferences.d/sid-100
```
```
apt-get update
apt-get install -t sid sshguard
```
#### Полезное
[Добавьте безопасные заголовки в nginx.conf](https://webdock.io/en/docs/how-guides/security-guides/how-to-configure-security-headers-in-nginx-and-apache)
[Тоже по заголовкам интересно](https://www.getpagespeed.com/server-setup/nginx-security-headers-the-right-way)
[Защитите свой домен (SPF)](https://dmarcian.com/domain-checker/) > [инструкция](https://www.cloudflare.com/learning/dns/dns-records/dns-spf-record/)
[Интересно по безопасности](https://owasp.org/www-community/attacks/csrf)
