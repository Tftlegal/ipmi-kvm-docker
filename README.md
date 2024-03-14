## Cobbler CentOS7
How to Install and Configure Cobbler on ECS CentOS 7
[Cobbler](https://www.alibabacloud.com/blog/how-to-install-and-configure-cobbler-on-ecs-centos-7_595708/)

## Hostkey INVAPI 
Реализация простой HTML5-панели управления серверами с поддержкой IPMI
[Hoskey wiki](https://hostkey.ru/blog/14-realizatsiya-prostoj-html5-paneli-upravleniya-serverami-s-podderzhkoj-ipmi/)

HOSTKEY INVAPI — сервисная панель, позволяющую выполнять любые действия по управлению оборудованием: от заказа серверов до переустановки ОС. Панель реализована в виде одностраничного веб-приложения, а все ее функции доступны также через API. Действия пользователя и запросы к API можно отслеживать в браузере через консоль разработчика (вызывается по Ctrl + Shift + I). Это позволяет анализировать все вызовы для отладки интеграции. 
21.04.2022
Реализация простой HTML5-панели управления серверами с поддержкой IPMI
server one
Содержание:

1. Проблемы традиционных решений
2. Внутреннее устройство INVAPI
3. Итоги
Для удаленного доступа к физическим серверам клиенты хостеров используют программные средства, которые работают только при наличии операционной системы и специального софта.

Если система не установлена или возникли какие-то проблемы с ее настройкой (например, при попытке изменить правила межсетевого экрана), доступ можно потерять. В таких ситуациях на помощь приходят специализированные контроллеры, позволяющие управлять серверами без ОС, как если бы вы сидели за физической консолью. Рассказываем, как это работает у нас в HOSTKEY.



Арендуйте выделенные и виртуальные серверы с моментальным деплоем в надежных дата-центрах класса TIER III в Москве и Нидерландах. Принимаем оплату за услуги HOSTKEY в Нидерландах в рублях на счет российской компании. Оплата с помощью банковских карт, в том числе и картой МИР, банковского перевода и электронных денег.
Проблемы традиционных решений
Обычным методом организации удаленного доступа к арендованным серверам считается управление на основе IPMI — запуск Java-плагина KVM.

Запуск Java-плагина KVM
* IPMI — интеллектуальный интерфейс управления платформой (отраслевой стандарт).

Для доступа к консоли нашим клиентам приходилось выполнять множество действий: активировать подключение, ждать, пока система выполнит проброс серого адреса, создавать временную учетную запись, переходить по ссылке с IP-адресом, авторизоваться в веб-интерфейсе и использовать интегрированный в сервер модуль IPMI.

Доступ к консолиДоступ к консоли
На клиентском устройстве требовалось установить программное обеспечение Java, что часто приводило к повышенной нагрузке на службу поддержки: не всем пользователям удавалось запустить скачанную консоль, были проблемы с версиями ПО, с запуском на Мас и т. д.

Эти недостатки стимулировали нас разработать более удобный и простой в обращении механизм управления оборудованием. Идея довольно очевидна: чтобы клиенту не пришлось устанавливать и настраивать софт, нужно сделать все на стороне хостера в безопасном виртуальном окружении.

Мы создали HOSTKEY INVAPI — сервисную панель, позволяющую выполнять любые действия по управлению оборудованием: от заказа серверов до переустановки ОС. Панель реализована в виде одностраничного веб-приложения, а все ее функции доступны также через API. Действия пользователя и запросы к API можно отслеживать в браузере через консоль разработчика (вызывается по Ctrl + Shift + I). Это позволяет анализировать все вызовы для отладки интеграции. Более подробно о концепции Invapi и процессе ее создания мы расскажем в отдельной статье.

Внутреннее устройство INVAPI
Чтобы избавить пользователей от лишних телодвижений, мы реализовали прямой вызов веб-консоли HTML5 из личного кабинета без локальной установки Java. Для практического воплощения идеи был использован Docker, а в основу решения легли сборки NoJava-IPMI-KVM-Server и ipmi-kvm-docker. Панель поддерживает материнские платы Supermicro до поколения 10X включительно, поскольку одиннадцатое поколение уже оснащено средством просмотра HTML5 Supermicro iKVM/IPMI.

Для доступа к консоли пользователю достаточно нажать кнопку:

Доступ к консоли
Активировать консоль также возможно с помощью запроса напрямую к API:

curl -s "https://invapi.hostkey.com/eq.php" -X POST \
--data "action=novnc" \
--data "token=SESSION_TOKEN" \
--data "id=SERVER_ID" \
--data "pin=PIN_CODE"
Пример ответа:

{
"result":"OK",
"scope":"http://rcnl1.hostkey.com:32800/vnc.html?host=IP ХОСТА&port=32800&autoconnect=true&password=YVhMxxhiuTpe3mH6y3ry",
"context":{"action":"novnc","id":"25250","location":"NL"},
"debug":"debug",
"key":"71ccb18b1fa499458526acc15fb6a40b"
}
Остается дождаться загрузки консоли — и можно работать, хотя внутреннее устройство процесса получения доступа выглядит сложнее. Для примера рассмотрим вызов HTML5-консоли для сервера с IPMI.

Общая схема вызова веб-консоли HTML5 из личного кабинета клиента:

Общая схема вызова веб-консоли HTML5
При запросе через INVAPI дается команда в API на открытие консоли для определенного сервера через кластер брокера сообщений (RabbitMQ). Для вызова консоли достаточно передать в брокер сообщений IP-адрес сервера и его локацию (наши серверы расположены в Нидерландах, США и России).

RabbitMQ передает данные сервера и задачу на открытие консоли созданному нашими специалистами вспомогательному сервису-ресиверу. Ресивер забирает данные, преобразует всю необходимую информацию, разделяет задачи (например, Cisco, IPMI и т. д.) и направляет их агентам.

Агенты (fence agents) соответствуют типам используемого в нашей инфраструктуре оборудования. Они обращаются на сервер с Docker-novnc, у которого есть доступ в закрытую сеть IPMI. Агент передает на сервер с Docker-novnc GET-запрос, в котором содержится IP-адрес и ID сервера, сессионный токен, а также ссылка на закрытие сеанса.

Структура запроса:

http://rcnl1.hostkey.com:ПОРТ/api/v1/server/IP_СЕРВЕРА/skey/КЛЮЧ_ЗАПРОСА/ID_СЕРВЕРА/closeurl/ССЫЛКА_НА_ЗАКРЫТИЕ
Содержание контейнера:

Содержание контейнера
Xvfb — X11 в виртуальном фрейм-буфере;
x11vnc — сервер VNC, который очищает указанный сервер X11;
noNVC — просмотрщик VNC на HTML5;
Fluxbox — оконный менеджер;
Firefox — браузер для просмотра консолей IPMI;
Java-плагин — Java требуется для доступа к большинству консолей IPMI KVM.
NoJava-IPMI-KVM-Server — это сервер на основе Python, позволяющий централизованно предоставлять sciapp/nojava-ipmi-kvm через браузер. Решение не требует установки Java или nojava-ipmi-kvm на локальных устройствах.

Ссылка для автоматического завершения сеанса добавлена нами для удобства пользователя и обеспечения безопасности оборудования: при отсутствии активности в течение определенного времени консоль будет закрыта автоматически. Этот вызов стартует сервис, запускающий контейнер с Docker-novnc, в котором содержится внешний IP-адрес для открытия консоли. Полное описание сборки и процесса установки контейнера NoJava-IPMI-KVM можно найти на GitHub.

Пример конфигурационного файла (~/.nojava-ipmi-kvmrc.yaml):

templates:
  kvm-openjdk-7u51:
    skip_login: False
    login_user: ADMIN
    login_endpoint: rpc/WEBSES/create.asp
    allow_insecure_ssl: False
    user_login_attribute_name: WEBVAR_USERNAME
    password_login_attribute_name: WEBVAR_PASSWORD
    send_post_data_as_json: False
    session_cookie_key: SessionCookie
    download_endpoint: Java/jviewer.jnlp
    java_version: 7u51
    format_jnlp: False
Запуск контейнера Docker:

usr/bin/nojava-ipmi-kvm -i 10.77.21.239 -u ADMIN -p PASSWD mykvmhost
[INFO] Check if 'http://10.77.21.239/' is reachable...
[INFO] The url 'http://10.77.21.239/' is reachable.
[INFO] Starting the Docker container...
[INFO] Waiting for the Docker container to be up and ready...
[INFO] Docker container is up and running.
[INFO] Url to view kvm console: http://IP_SERV:ID_SERV/vnc.html?host=IP_SERV&port=32769&autoconnect=true&password=PASSWD
http://IP_SERV:ID_SERV/vnc.html?host=IP_SERV&port=ID_SERV&autoconnect=true&password=PASSWD
Запуск контейнера Docker
Скрипт для запуска сервиса:

#/bin/python3
# EASY-Install-Entry_Script: 'nojava-ipmi-kvm==0.9.0', 'console_scripts', 'nojava-ipmi-kvm'
__requires__ = 'nojava-ipmi-kvm==0.9.0'
import re
import sys
from pkg_resources import load_entry_point

if __name__ == ' __main__ ':
    sys.argv[0] = re.sub(r'(-script\.pyw?|\.exe)?$', '', sys.argv[0])
    sys.exit(
        load_entry_point('nojava-ipmi-kvm==0.9.0', 'console_scripts', 'nojava-ipmi-kvm')()
    )
Итоги
Внедрение нового решения значительно упростило для конечных пользователей процесс управления оборудованием Supermicro, а также снизило нагрузку на нашу службу поддержки. Для серверов с доступом по протоколу VNC мы реализовали консоль HTML5 с помощью Apache Guacamole, что также позволило упростить управление «железом» других производителей.

Кстати, в нашей панели управления серверами HOSTKEY кроме описанных возможностей планируется расширение функциональности. Если вас интересуют дополнительные функции и особенности работы панели или нашего API – пишите в комментариях.


## Ipmi-kvm-docker
![Docker Image Size (tag)](https://img.shields.io/docker/image-size/solarkennedy/ipmi-kvm-docker/latest)
![Docker Pulls](https://img.shields.io/docker/pulls/solarkennedy/ipmi-kvm-docker)

Ever wanted to access and IPMI KVM console, only to find that you don't
have network access or the right version of java or a compatible
browser or credentials?

This container runs:

* Xvfb - X11 in a virtual framebuffer
* x11vnc - A VNC server that scrapes the above X11 server
* [noNVC](https://kanaka.github.io/noVNC/) - A HTML5 canvas vnc viewer
* Fluxbox - a small window manager
* Firefox - For browsing IPMI consoles
* Java-plugin - Because... you need java to access most IPMI KVM Consoles.

This is a [trusted build](https://registry.hub.docker.com/u/solarkennedy/ipmi-kvm-docker/)
on the Docker Hub.

## Run It

    # on a remote host that can reach ipmi
    ssh admin
    $ docker run -p 8080:8080 solarkennedy/ipmi-kvm-docker

    # Now on your laptop
    xdg-open http://admin:8080
    # On a mac
    open http://admin:8080
    # Or just open in a browser

In your web browser you should see the firefox, ready to connect to
and IPMI KVM:

![IPMI Screenshot](https://raw.githubusercontent.com/solarkennedy/ipmi-kvm-docker/master/screenshot.png)

### Custom resolution

By default, the VNC session will run with a resolution of 1024x768 (with 24-bit color depth).
Custom resolutions can be specified with the docker environment variable RES, and must include color depth.

    $ docker run -p 8080:8080 -e RES=1600x900x24 solarkennedy/ipmi-kvm-docker

### Mount volume

In case you need to mount floppy/iso images to the machine you can mount a volume to the container.

    $ docker run -p 8080:8080 -v /your/local/folder:/root/images solarkennedy/ipmi-kvm-docker
