# Репозиторий предназначен для настройки маршрутизации через прозрачный прокси на openwrt 22+, в которых по умолчанию используется nftables
## Настройка роутера на OpenWrt на точечную маршрутизацию по доменам, с использованием прозрачного прокси
На основе идеи по [точечной маршрутизации от itdog](https://itdog.info/tochechnaya-marshrutizaciya-po-domenam-na-routere-s-openwrt/) решил сделать свой вариант маршрутизации трафика по доменам через прозрачный прокси.
### Для работы nftset нужно установить dnsmasq-full
```shell
opkg update && cd /tmp/ && opkg download dnsmasq-full
opkg remove dnsmasq && opkg install dnsmasq-full --cache /tmp/
mv /etc/config/dhcp-opkg /etc/config/dhcp
/etc/init.d/dnsmasq restart
```
### Для ускорения ютуба и других сервисов с неисправными cdn. Будем использовать проект [hufrea/byedpi](https://github.com/hufrea/byedpi) в режиме прозрачного прокси
Установим инструмент
на странице [Releases](https://github.com/hufrea/byedpi/releases) выберите нужную вам архитектуру и скопируйте ссылку на файл
```shell
wget https://github.com/hufrea/byedpi/releases/download/v0.16.6/byedpi-16.6-aarch64.tar.gz
tar -xvf byedpi-16.6-aarch64.tar.gz
rm byedpi-16.6-aarch64.tar.gz
```
Запустим, как фоновый процесс:
```shell
./ciadpi-aarch64 -s1 -o1 -Ar -o1 -At - f-1 -r1+s -As -E &
```
Первый раз лучше запускать `./ciadpi-aarch64 -s1 -o1 -Ar -o1 -At - f-1 -r1+s -As` и на клиентском устройстве попробывать подключится по socks5 протоколу, по умолчанию порт 1080, `socks5://ip-router:1080`
### Создание файла с цепочкой маршрутизации для nftables
Например файл /root/byedpi.nft
```shell
set proxylist {
    type ipv4_addr
    flags interval
    timeout 4h
}

chain byedpi_prerouting {
    type nat hook prerouting priority -100;
    ip daddr @proxylist tcp dport {80, 443} redirect to 1080
}
```

### Настройка firewall в openwrt
В файле `/etc/config/firewall` нужно добавить
```shell
config include
        option enabled '1'
        option type 'nftables'
        option path 'your-script-path'
        option position 'table-post'
```
#### Вам необходимо изменить `your-script-path` на ваш путь до файла с правилами nftables. В моём случае путь будет /root/byedpi.nft
Теперь перезапустим firewall и убедимся, что он рабоет и не возвращает ошибок
```shell
/etc/init.d/firewall restart
```

### Добавление новой папки конфигурации для dnsmasq в OpenWRT:
#### Вариант №1 Через добавление параметра в /etc/config/dhcp
```shell
config dnsmasq
        option confdir '/etc/dnsmasq.d'
```
После этого нужно перезапустить dnsmasq и убедиться, что он работает
```
/etc/init.d/dnsmasq restart
/etc/init.d/dnsmasq status
```
Если статус `not running`, то лучше попробовать вариант 2 

#### Вариант №2 Через uci
```shell
uci set dhcp.@dnsmasq[0].confdir='/etc/dnsmasq.d'
uci commit dhcp
/etc/init.d/dnsmasq restart
```
### Добавление списка доменов, которые будут идти через прозрачный прокси
Файл /etc/dnsmasq.d/proxylist.conf
Например для ускорения youtube
```shell
#youtube
nftset=/returnyoutubedislikeapi.com/4#inet#fw4#proxylist
nftset=/youtube-nocookie.com/4#inet#fw4#proxylist
nftset=/youtube-ui.l.google.com/4#inet#fw4#proxylist
nftset=/youtube.com/4#inet#fw4#proxylist
nftset=/youtubeembeddedplayer.googleapis.com/4#inet#fw4#proxylist
nftset=/youtubei.googleapis.com/4#inet#fw4#proxylist
nftset=/youtubekids.com/4#inet#fw4#proxylist
nftset=/wide-youtube.l.google.com/4#inet#fw4#proxylist
nftset=/yt-video-upload.l.google.com/4#inet#fw4#proxylist
nftset=/ytimg.com/4#inet#fw4#proxylist
nftset=/ytimg.l.google.com/4#inet#fw4#proxylist
nftset=/yting.com/4#inet#fw4#proxylist
nftset=/youtu.be/4#inet#fw4#proxylist
nftset=/googlevideo.com/4#inet#fw4#proxylist
nftset=/ggpht.com/4#inet#fw4#proxylist
nftset=/suggestqueries.google.com/4#inet#fw4#proxylist
nftset=/noifications-pa.googleapis.com/4#inet#fw4#proxylist
nftset=/googleusercontent.com/4#inet#fw4#proxylist
```
Перезапустим dnsmasq, чтобы правила применились
```shell
/etc/init.d/dnsmasq restart
```
Роутер настроен, теперь нужно убедиться что ваше устройство использует роутер, как dns серевер (Иначе работать не будет)
