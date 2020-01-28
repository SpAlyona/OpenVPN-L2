Настройка OpenVPN L2. Ubuntu 16.04
Установка OpenVPN
Сначала установим OpenVPN на наш сервер. Также мы установим пакет easy-rsa, который позволит нам настроить наш собственный внутренний центр сертификации (certificate authority, CA) для использования с нашей VPN.
Обновим список пакетов сервера и установим необходимые пакеты следующими командами:

```
sudo apt update
sudo apt install openvpn easy-rsa
sudo apt install bridge-utils
```

Создание директории центра сертификации
OpenVPN это виртуальная частная сеть, использующая TLS/SSL. Это означает, что OpenVPN использует сертификаты для шифрования трафика между сервером и клиентами. Для выпуска доверенных сертификатов (trusted certificates) нам потребуется создать наш собственный центр сертификации.
Для начала копируем шаблонную директорию easy-rsa в нашу домашнюю директорию с помощью команды make-cadir:

```
make-cadir ~/openvpn-ca
```

Далее зайдём в эту директорию для начала настройки центра сертификации:

```
cd ~/openvpn-ca
```

Настройка переменных центра сертификации
Для настройки переменных нашего центра сертификации нам необходимо отредактировать файл vars. Откройте этот файл в вашем текстовом редакторе:

```
nano vars
```

Внутри файла вы найдёте переменные, которые можно отредактировать, и которые задают параметры сертификатов при их создании. Нам нужно изменить всего несколько переменных.
Перейдите ближе к концу файла и найдите настройки полей, используемые по умолчанию при создании сертификатов. Они должны выглядеть примерно так:

```
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="SanFrancisco"
export KEY_ORG="Fort-Funston"
export KEY_EMAIL="me@myhost.mydomain"
export KEY_OU="MyOrganizationalUnit"
```

Замените значения, выделенные красным, на что-нибудь другое, не оставляйте их незаполненными:

```
export KEY_COUNTRY="RU"
export KEY_PROVINCE="Moscow"
export KEY_CITY="Moscow"
export KEY_ORG="MireaMoscowOrg"
export KEY_EMAIL="my@mail.my"
export KEY_OU="Community"
```

Пока мы в этом файле, отредактируем значение KEY_NAME чуть ниже, которое заполняет поле субъекта сертификатов. Для простоты зададим ему название server:

```
export KEY_NAME="server"
```

Сохраните и закройте файл.
Создание центра сертификации
Теперь мы можем использовать заданные нами переменные и утилиты easy-rsa для создания центра сертификации.
Убедитесь, что вы находитесь в директории центра сертификации и используйте команду source к файлу vars:

```
cd ~/openvpn-ca
source vars
```

Вы должны увидеть следующий вывод:

```
NOTE: If you run ./clean-all, I will be doing a rm -rf on /home/sammy/openvpn-ca/keys
```

Убедимся, что мы работаем в “чистой среде” выполнив следующую команду:

```
./clean-all
```

Теперь мы можем создать наш корневой центр сертификации командой:

```
./build-ca
```

Эта команда запустит процесс создания ключа и сертификата корневого центра сертификации. Поскольку мы задали все переменные в файле vars, все необходимые значения будут введены автоматически. Нажимайте ENTER для подтверждения выбора. Вывод примерно такой:

```
Generating a 2048 bit RSA private key
..........................................................................................+++
...............................+++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [RU]:
State or Province Name (full name) [Moscow]:
Locality Name (eg, city) [Moscow]:
Organization Name (eg, company) [MireaMoscowOrg]:
Organizational Unit Name (eg, section) [Community]:
Common Name (eg, your name or your server's hostname) [Mirea CA]:
Name [server]:
Email Address [my@mail.my]:
```

Теперь у нас есть центр сертификации, который мы сможем использовать для создания всех остальных необходимых нам файлов.
Создание сертификата, ключа и файлов шифрования для сервера
Далее создадим сертификат, пару ключей и некоторые дополнительные файлы, используемые для осуществления шифрования, для нашего сервера.
Начнём с создания сертификата OpenVPN и ключей для сервера. Это можно сделать следующей командой:
Внимание: Если ранее вы выбрали имя, отличное от server, вам придётся немного изменить некоторые инструкции. Например, при копировании созданных файлов в директорию /etc/openvpn вам придётся заменить имена на заданные вами. Вам также придётся изменить файл /etc/openvpn/server.conf для того, чтобы он указывал на корректные .crt и .key файлы.
Выполним:

```
./build-key-server server
```

Вывод опять будет содержать значения по умолчанию, переданные этой команде (server), а также значения из файла vars.
Согласитесь со всеми значениями по умолчанию, нажимая ENTER. Не задавайте challenge password! В конце процесса два раза введите y для подписи и подтверждения создания сертификата:
Вывод:

```
Certificate is to be certified until May  1 17:51:16 2026 GMT (3650 days)
Sign the certificate? [y/n]:y
1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

Далее создадим оставшиеся файлы. Мы можем сгенерировать сильные ключи протокола Диффи-Хеллмана, используемые при обмене ключами, командой:

```
./build-dh
```

Для завершения этой команды может потребоваться несколько минут.
Далее мы можем сгенерировать подпись HMAC для усиления способности сервера проверять целостность TSL:

```
openvpn --genkey --secret keys/ta.key
```

Так как мы работаем с настройкой OpenVPN L2, то нам необходимо сконфигурировать статический ключ. У статического ключа всегда 1 клиент и 1 сервер. Он должен существовать как открытый текст на каждом узле VPN.
Создайте статический ключ:

```
openvpn --genkey --secret static.key
```

Создание сертификата и пары ключей для клиента
Далее мы можем сгенерировать сертификат и пару ключей для клиента. Вообще это можно сделать и на клиентской машине и затем подписать полученный ключ центром сертификации сервера, но в этой статье для простоты мы сгенерируем подписанный ключ на сервере.
Поскольку мы можем вернуться к этому шагу позже, мы повторим команду source для файла vars. Мы будем использовать параметр client1 для создания первого сертификата и ключа.
Для создания файлов без пароля для облегчения автоматических соединений используйте команду build-key:

```
cd ~/openvpn-ca
source vars
./build-key client1
```

Для создания файлов, защищённых паролем, используйте команду build-key-pass:

```
cd ~/openvpn-ca
source vars
./build-key-pass client1
```

В ходе процесса создания файлов все значения по умолчанию будут введены, вы можете нажимать ENTER. Не задавайте challenge password и введите y на запросы о подписи и подтверждении создания сертификата.
Настройка сервиса OpenVPN
Далее настроим сервис OpenVPN с использованием созданных ранее файлов.
Копирование файлов в директорию OpenVPN
Нам необходимо скопировать нужные нам файлы в директорию /etc/openvpn.
Сначала скопируем созданные нами файлы. Они находятся в директории ~/openvpn-ca/keys, в которой они и были созданы. Нам необходимо скопировать сертификат и ключ центра сертификации, сертификат и ключ сервера, подпись HMAC и файл Diffie-Hellman:

```
cd ~/openvpn-ca/keys
sudo cp ca.crt ca.key server.crt server.key ta.key dh2048.pem /etc/openvpn
```

Далее нам необходимо скопировать и распаковать файл-пример конфигурации OpenVPN в конфигурационную директорию, мы будем использовать этот файл в качестве базы для наших настроек:
gunzip -c /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz | sudo tee /etc/openvpn/server.conf
Настройка конфигурации OpenVPN
Теперь, когда наши файлы находятся на своём месте, займёмся настройкой конфигурационного файла сервера:

```
sudo nano /etc/openvpn/server.conf
```

Содержание конфига должно быть таким:

```
local 192.168.11.2              #физический адрес интерфеса eth0
port 1194
proto udp
dev tap0
secret static.key
push “redirect-gateway def1 bypass-dhcp”
push “dhcp-option DNS 8.8.8.8”
push “dhcp-option DNS 8.8.4.4”
keepalive 10 120
cipher AES-128-CBC
auth SHA256
comp-lzo
user nobody
group nogroup
persist-key
persist-tun
status openvpn-status.log
verb 3
```

Сохраните и закройте файл.
Настройка сетевой конфигурации сервера
Далее нам необходимо настроить сетевую конфигурацию сервера, чтобы OpenVPN мог корректно перенаправлять трафик.
Настройка перенаправления IP
Сначала разрешим серверу перенаправлять трафик. Это ключевая функциональность нашего VPN сервера.
Настроим это в файле /etc/sysctl.conf:

```
sudo nano /etc/sysctl.conf
```

Найдите строку настройки net.ipv4.ip_forward. Удалите “#” из начала строки, чтобы раскомментировать её:

```
net.ipv4.ip_forward=1
```

Сохраните и закройте файл.
Для применения настроек к текущей сессии наберите команду:

```
sudo sysctl -p
```

Настройка правил UFW для сокрытия соединений клиентов

Далее мы должны найти публичный интерфейс сети (public network interface). Для этого наберите команду:

```
ip route | grep default
```

Публичный интерфейс должен следовать за словом “dev”. Например, в нашем случае этот интерфейс называется eth0:
Вывод
![p3](https://sun9-12.userapi.com/c206616/v206616376/4dfad/-dmyp6T_e9I.jpg)

Зная название интерфейса откроем файл /etc/ufw/before.rules и добавим туда соответствующие настройки:

```
sudo nano /etc/ufw/before.rules
```

Это файл содержит настройки UFW, которое применяются перед применением правил UFW. Добавьте в начало файла выделенные красным строки. Это настроит правила, применяемые по умолчанию, к цепочке POSTROUTING в таблице nat и будет скрывать весь трафик от VPN:
Внимание: не забудьте заменить eth0 в строке -A POSTROUTING на имя интерфейса, найденное нами ранее.

```
#
# rules.before
#
# Rules that should be run before the ufw command line added rules. Custom
# rules should be added to one of these chains:
#   ufw-before-input
#   ufw-before-output
#   ufw-before-forward
#

# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0] 
# Allow traffic from OpenVPN client to eth0
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES

# Don't delete these required lines, otherwise there will be errors
*filter
```

Сохраните и закройте файл.
Теперь мы должны сообщить UFW, что ему по умолчанию необходимо разрешать перенаправленные пакеты. Для этого откройте файл /etc/default/ufw:

```
sudo nano /etc/default/ufw
```

Найдите в файле директиву DEFAULT_FORWARD_POLICY. Мы изменим значение с DROP на ACCEPT:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Сохраните и закройте файл.
Открытие порта OpenVPN и применение изменений
Далее настроим сам файрвол для разрешения трафика в OpenVPN.
Если вы не меняли порт и протокол в файле /etc/openvpn/server.conf, вам необходимо разрешить трафик UDP для порта 1194. Если вы изменили эти настройки, введите указанные вами значения.
Даже мы добавим порт SSH на случай, если вы не сделали этого ранее.

```
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
Теперь деактивируем и активируем UFW для применения внесённых изменений:
sudo ufw disable
sudo ufw enable
```

Теперь наш сервер сконфигурирован для обработки трафика OpenVPN.
Включение сервиса OpenVPN
Мы готовы включит сервис OpenVPN на нашем сервере. Мы можем сделать это с помощью systemd.
Нам необходимо запустить сервер OpenVPN указав имя нашего файла конфигурации в качестве переменной после имени файла systemd. Файл конфигурации для нашего сервера называется /etc/openvpn/server.conf, поэтому мы добавим @server в конец имени файла при его вызове:

```
sudo systemctl start openvpn@server
ip addr show tap0
```

Если всё получилось, вывод должен выглядеть примерно следующим образом:
![p3](https://sun9-12.userapi.com/c206616/v206616376/4dfad/-dmyp6T_e9I.jpg)


Если всё в порядке, настроем сервис на автоматическое включение при загрузке сервера:
sudo systemctl enable openvpn@server
Создание инфраструктуры настройки клиентов
Далее настроим систему для простого создания файлов конфигурации для клиентов.
Создание структуры директорий конфигурации клиентов
В домашней директории создайте структуру директорий для хранения файлов:

```
mkdir -p ~/client-configs/files
```

Поскольку наши файлы конфигурации будут содержать клиентские ключи, мы должны настроить права доступа для созданных директорий:

```
chmod 700 ~/client-configs/files
```

Создание базовой конфигурации
Далее скопируем конфигурацию-пример в нашу директорию для использования в качестве нашей базовой конфигурации:

```
cp /usr/share/doc/openvpn/examples/sample-config-files/static-home.conf ~/client-configs/base.conf
```

Откройте этот файл в вашем текстовом редакторе:

```
nano ~/client-configs/base.conf
```

Сначала найдите директиву remote. Эта директива сообщает клиенту адрес нашего сервера OpenVPN. Это должен быть публичный IP адрес вашего сервера OpenVPN. Если вы изменили порт, который слушает сервер OpenVPN, измените порт по умолчанию 1194 на ваше значение:

```
# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.
dev tap
proto udp
ifconfig 10.1.0.2 10.1.0.1
secret static.key
port 1194
remote 192.168.11.2 1194
user nobody
group nogroup
verb 3
key-direction 1
cipher AES-128-CBC
auth SHA256
```

Наконец, добавьте несколько закомментированных строк. Мы ходим добавить эти строки в каждый файл конфигурации, но они будут включены только для клиентов на Linux, которые используют файл /etc/openvpn/update-resolv-conf. Этот скрипт использует утилиту resolvconf для обновление информации DNS на клиентах Linux.

```
# script-security 2
# up /etc/openvpn/update-resolv-conf
# down /etc/openvpn/update-resolv-conf
```

Если ваш клиент работает на Linux и использует файл /etc/openvpn/update-resolv-conf, вы должны раскомментировать эти строки в сгенерированном клиентском файле конфигурации OpenVPN.
Сохраните и закройте файл.
Создание скрипта генерации файлов конфигурации
Теперь создадим простой скрипт для генерации файлов конфигурации с релевантными сертификатами, ключами и файлами шифрования. Он будет помещать сгенерированные файла конфигурации в директорию ~/client-configs/files.
Создайте и откройте файл make_config.sh внутри директории ~/client-configs:

```
nano ~/client-configs/make_config.sh
```

Вставьте следующие текст в этот файл:

```
#!/bin/bash
# First argument: Client identifier
KEY_DIR=~/openvpn-ca/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf
cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.conf
```

Сохраните и закройте файл.

Сделайте его исполняемым файлом командой:

```
chmod 700 ~/client-configs/make_config.sh
```

Вводим команду:

```
ifconfig 
```

Мы должны увидеть поднятый tap0-интерфейс.
Теперь настроим виртуальный мост.

```
brctl addbr br0
brctl addif br0 tap0
brctl addif br0 eth0
ip link set tap0 up
ip link set br0 up
```

Генерация конфигураций клиентов
Теперь мы можем легко сгенерировать файлы конфигурации клиентов.
Если вы следовали всем шагам этой статьи, вы создали сертификат client1.crt и ключ клиента client1.key командой ./build-key client1 на шаге 6. Вы можете сгенерировать конфигурацию для этих файлов пе
Vsрейдя в директорию ~/client-configs и используя только что созданный нами скрипт:

```
cd ~/client-configs
./make_config.sh client1
```

Если всё прошло успешно, мы должны получить файл client1.conf в директории ~/client-configs/files:

```
ls ~/client-configs/files
```
Вывод

```
client1.conf
```

Доставка конфигураций клиентам
Ниже мы приводим пример передачи файла client1.conf с использованием SFTP. Следующую команду можно использовать на вашем компьютере -клиенте:

```
sftp sammy@openvpn_server_ip:client-configs/files/client1.ovpn ~/etc/openvpn/
```

Установка файлов конфигураций клиентов
На компьютере-клиенте(Ubuntu 16.04)

```
sudo apt-get update
sudo apt-get install openvpn
sudo apt install bridge-utils
sftp sammy@openvpn_server_ip:client-configs/files/client1.conf ~/
```

Сначала проверьте, содержит ли ваш дистрибутив скрипт /etc/openvpn/update-resolv-conf:

```
ls /etc/openvpn
```

Вывод

```
update-resolve-conf
```

Далее отредактируйте полученный с сервера файл конфигурации клиента OpenVPN:

```
nano /etc/openvpn/client1.conf
```

Если вам удалось найти файл update-resolv-conf, раскомментируйте следующие строки файла:

```
script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
```

Сохраните и закройте файл.
Теперь вы можете соединиться с VPN используя команду openvpn следующим образом:

```
systemctl start openvpn@client1
```

В результате вы подключитесь к серверу.
Настроим виртуальный мост на клиенте:

```
brctl addbr br0
brctl addif br0 tap0
brctl addif br0 eth0
ip link set tap0 up
ip link set br0 up
```
