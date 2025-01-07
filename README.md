# Debian-iPXE Сетевая установка Debian Bookworm из локального репозитория созданного из iso образа DVD диска.
 Не претендую на первоисточник, но в интернете не нашел полноценной инструкции для развертывания сервера iPXE для установки Debian Bookworm из репозитория в локальной сети.
 Главное условие, репозиторий должен быть создан из iso образа DVD диска.
 Мой стенд, Proxmox -> изолированная сеть -> Виртуальные машины 
 1. Proxy Squid - для доступа к интернету за пределами тестового стенда.
 2. Непосредственно тестовый сервер для iPXE.
    
 Установка iPXE будет поддерживать как EFI так и BIOS Legacy
 Устанавливаем на виртуальныю машину Debian Bookworm c iso образа https://cdimage.debian.org/cdimage/release/current/amd64/iso-dvd/debian-12.8.0-amd64-DVD-1.iso
 Далее этот образ нам понадобится для создания нашего репозитория.
 В данной инстуркции будут использованы 2 ip адреса
 192.168.2.2 - прокси сервер для доступа в интернет, необходим для установки пакетов и обновлений во время настройки ipxe.
 192.168.2.4 - непосредственно наш будущий iPXE сервер
 Приступим к настройке iPXE на нашей чситой установки Debian Bookworm!

```
1)Конфигурация сетевого интерфейса. У меня enp6s18, для просмотра своего выполните ip a или ip l
cat > /etc/network/interfaces <<EOF
#This file describes the network interfaces available on your system1
#and how to activate them. For more information, see interfaces(5).
source /etc/network/interfaces.d/*
#The loopback network interface
auto lo
iface lo inet loopback
#The primary network interface для просмотра имени интерфейса ip a
allow-hotplug enp6s18
iface enp6s18 inet static
address 192.168.2.4
netmask 255.255.255.0
gateway 192.168.2.2
EOF
###########################################################################
2) Далее прописываем настройки нашего прокси для apt
cat > \etc\apt\apt.conf <<EOF
Acquire::http::proxy "http://192.168.2.2:3128";
Acquire::https::proxy "http://192.168.2.2:3128";
EOF

#Далее настройки прокси для системы для всех пользователей.
cat > /etc/profile.d/proxy.sh <<EOF
#http/https
export http_proxy="http://192.168.2.2:3128/"
export https_proxy="http://192.168.2.2:3128/"
EOF
#Делаем файл proxy.sh исполняемым
chmod +x /etc/profile.d/proxy.sh
#Запускаем proxy.sh
. /etc/profile.d/proxy.sh
#Проверяем системные прокси
env | grep -i proxy
########################################################################
3)Правим списки репозиториев, добавляем репы яндекса для книжного червя
cat >/etc/apt/sources.list<<EOF
deb http://mirror.yandex.ru/debian/ bookworm main
deb-src http://mirror.yandex.ru/debian/ bookworm main
deb http://mirror.yandex.ru/debian-security bookworm-security main contrib
deb-src http://mirror.yandex.ru/debian-security bookworm-security main contrib
deb http://mirror.yandex.ru/debian/ bookworm-updates main contrib
deb-src http://mirror.yandex.ru/debian/ bookworm-updates main contrib
EOF
#запускаем 
apt update  
##################################################################
4) Установка необходимого ПО для iPXE
apt install -y dnsmasq nfs-kernel-server rsync git gcc make perl liblzma-dev mtools apache2 reprepro gpg
###########################################################
5) создание каталогов для ipxe/tftp+nfs+apache
mkdir -p /srv/share/{ipxe,nfs,www}
#настройка NFS Добавляем конфигурацию в конец файла /etc/exports" NFS добавил для будущей поддержки других дистрибутивов Linux.
echo "/srv/share/nfs *(rw,sync,no_wdelay,insecure_locks,no_root_squash,insecure,no_subtree_check)" >> /etc/exports
#Перезапуск демона nfs-kernel-server или можно выполнить exportfs -av
systemctl restart nfs-kernel-server
############################################################
6)Конфигурация файла  /etc/dnsmasq.conf interface=enp6s18 необходимо указать актуальный сетевой интерфейс, посмотреть можно выполнив ip a или ip l
cp /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
cat > /etc/dnsmasq.conf <<EOF
interface=enp6s18
bind-interfaces
domain=workgroup
dhcp-range=192.168.2.10,192.168.2.20,255.255.255.0,1h
dhcp-option=option:router,192.168.2.4
dhcp-option=option:dns-server,192.168.2.4
dhcp-leasefile=/var/lib/misc/dnsmasq.leases
enable-tftp
tftp-root=/srv/share/ipxe
#boot config for BIOS systems
dhcp-match=set:bios-x86,option:client-arch,0
dhcp-boot=tag:bios-x86,firmware/ipxe.pxe
#boot config for UEFI systems
dhcp-match=set:efi-x86_64,option:client-arch,7
dhcp-match=set:efi-x86_64,option:client-arch,9
dhcp-boot=tag:efi-x86_64,firmware/ipxe.efi
EOF
#Перезапускаем dnsmasq
systemctl restart dnsmasq
##############################################################
7)настройка apache2
cat > /etc/apache2/sites-available/debpxe.conf <<EOF
<VirtualHost *:80>
  ServerAdmin webmaster@localhost
  DocumentRoot /srv/share/www
    <Directory /srv/share/www>
     Options Indexes FollowSymLinks
     AllowOverride None
     Require all granted
    </Directory>
  ErrorLog ${APACHE_LOG_DIR}/DebPXEerror.log
  CustomLog ${APACHE_LOG_DIR}/DebPXEaccess.log combined
</VirtualHost>
EOF
#отключение дефолтной и включение нашей конфигурации, перезапуск apache2
#Выключайм конфигурацию по умолчанию
a2dissite 000-default.conf 
#включаем нашу конфигурацию (если ошибка то создаем ссылку ln -s /etc/apache2/sites-available/debpxe.conf  /etc/apache2/sites-enabled/debpxe.conf после a2ensite debpxe.conf)
a2ensite debpxe.conf 
systemctl reload apache2
#для проверки синтаксиса нашего файла конфигурации выполнить apache2ctl -t
##########################################################
8) Установка и настройка iPXE. Клонируем проект с github
#т.к. каталог для tftp /srv/share/ipxe создаем каталоги конфигурации для ipxe
mkdir -p /srv/share/ipxe/{config,firmware}
mkdir /srv/tmp
cd /srv/tmp
git clone https://github.com/ipxe/ipxe.git
cd /srv/tmp/ipxe/src
#создаем скрипт
cat > /srv/tmp/ipxe/src/bootconfig.ipxe <<EOF
#!ipxe
dhcp
chain tftp://192.168.2.4/config/boot.ipxe
EOF

#делаем бэкапы и редактируем файлы /srv/tmp/ipxe/src/config/general.h и  /srv/tmp/ipxe/src/config/console.h
cp /srv/tmp/ipxe/src/config/console.h /srv/tmp/ipxe/src/config/console.h.backup
cp /srv/tmp/ipxe/src/config/general.h /srv/tmp/ipxe/src/config/general.h.backup
#раскоментируем строки для включения необходимых функций для iPXE в файлах general.h и console.h
#define DOWNLOAD_PROTO_TFTP  
#define DOWNLOAD_PROTO_HTTP      
#define PING_CMD
#define IPSTAT_CMD
#define REBOOT_CMD
#define POWEROFF
#define CONSOLE_CMD
#undef  DOWNLOAD_PROTO_NFS
#define       CONSOLE_FRAMEBUFFER

sed -i 's/#undef.*DOWNLOAD_PROTO_NFS/#define	DOWNLOAD_PROTO_NFS/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*DOWNLOAD_PROTO_HTTP/#define DOWNLOAD_PROTO_HTTP/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*DOWNLOAD_PROTO_TFTP/#define DOWNLOAD_PROTO_TFTP/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*PING_CMD/#define PING_CMD/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*IPSTAT_CMD/#define IPSTAT_CMD/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*REBOOT_CMD/#define REBOOT_CMD/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*POWEROFF_CMD/#define POWEROFF_CMD/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*CONSOLE_CMD/#define CONSOLE_CMD/' /srv/tmp/ipxe/src/config/general.h
sed -i 's/\/\/#define.*CONSOLE_FRAMEBUFFER/#define	CONSOLE_FRAMEBUFFER/' /srv/tmp/ipxe/src/config/console.h


#Далее запускаем компиляцию файлов. Много файлов
cd /srv/tmp/ipxe/src
make bin/ipxe.pxe bin/undionly.kpxe bin/undionly.kkpxe bin/undionly.kkkpxe bin-x86_64-efi/ipxe.efi EMBED=bootconfig.ipxe
#Копируем файлы для ipxe
cp -v bin/{ipxe.pxe,undionly.kpxe,undionly.kkpxe,undionly.kkkpxe} bin-x86_64-efi/ipxe.efi /srv/share/ipxe/firmware
######################################################################
9) работа с ISO,пакетами для репозитория и подпиью
# необходим iso образ, создать подпись для репозитория,
# создаем каталог для хранения пакетов для перозитория.
mkdir -p /srv/share/www/debian12/{cache,conf,repo}
# копируем образ с другово ПК (у меня лежит на 192.168.2.2) себе в srv
scp root@192.168.2.2:/srv/apache2/iso/debian-12.8.0-amd64-DVD-1.iso /srv
# создаем каталог для монтирования и монтируем образ
mkdir /mnt/cdrom
mount -o loop /srv/debian-12.8.0-amd64-DVD-1.iso /mnt/cdrom
# Копируем пакеты .deb и .udeb во временный каталог  
find /mnt/cdrom/pool -type f -name "*.deb" -print -exec cp {} /srv/share/www/debian12/cache \;
find /mnt/cdrom/pool -type f -name "*.udeb" -print -exec cp {} /srv/share/www/debian12/cache \;
# создаем подпись для репозитория  указываем пароль qwert отвечаем на кучу вопросов и получаем заветный key-id который копируем в последнюю строку файла SignWith: 28E9A2D344FF7A86082A56616195B3635B1275C8
# создание с диалогами gpg --full-generate-key
# создание пары ключей без диалогов
gpg --batch --pinentry-mode=loopback --passphrase "qwert" --quick-generate-key "Debian repository <ipxe@mail.ru>" rsa4096 default none
# gpg --list-public-keys - посмотреть публичные ключи.
# gpg --list-secret-keys - посмотреть секретные ключи
# gpg --delete-secret-and-public-key ID-KEY - удаляет закрытый и открытый ключ по его ID
# apt-key list - список ключей установленных в системе
# экспорт открытого ключа из закрытого ключа по его ID, ID можно узнать предыдущими закомментироваными командами
gpg --output /srv/share/www/debian12/public.pgp --export 28E9A2D344FF7A86082A56616195B3635B1275C8
#запоминаем пароль и ID ключа, необходимо для пункта 11 данной инструкции.
###########################################################################
10)Работа с initrd
#создаем каталоги для хранения initrd образов
mkdir -p /srv/init/{dvd,netboot,dvdz,netbootz}
mkdir /srv/share/www/debian12/netboot
cd /srv/share/www/debian12/netboot
#загрузка netboot gtk с красивой графикой и распаковка
wget http://ftp.debian.org/debian/dists/bookworm/main/installer-amd64/20230607+deb12u8/images/netboot/gtk/netboot.tar.gz
tar -xzvf netboot.tar.gz
mv /srv/share/www/debian12/netboot/netboot.tar.gz /srv/init/
cp /srv/share/www/debian12/netboot/debian-installer/amd64/initrd.gz /srv/init/netboot/
cd /srv/init/netboot/
gzip -dc initrd.gz | cpio -i
mv /srv/init/netboot/initrd.gz /srv/init/initrd.gz.netboot
#копируем initrd с ISO с графическим интерфейсом и распаковка
cp /mnt/cdrom/install.amd/gtk/initrd.gz /srv/init/dvd
cd /srv/init/dvd
gzip -dc initrd.gz | cpio -i
mv /srv/init/dvd/initrd.gz /srv/init/initrd.gz.dvd
#копирование драйверов из образа initrd ISO в образ initrd netboot
#Далее сравниваем список драйверов, если есть желание.
ls /srv/init/dvd/lib/modules/6.1.0-27-amd64/kernel/drivers/
ls /srv/init/netboot/lib/modules/6.1.0-27-amd64/kernel/drivers/
#что надо копируем, если не определяются диски то копируем драйвера из initrd находящегося на dvd в initrd из netboot
#Присмотреться к /srv/init/dvd/lib/modules/6.1.0-27-amd64/kernel/drivers/scsi/ посмотреть содержимое в обоих initrd.gz
#Присмотреться к /srv/init/dvd/lib/modules/6.1.0-27-amd64/kernel/fs посмотреть содержимое в обоих initrd.gz
#следующие команды скопируют необходимое из initrd DVD на initrd из netboot
cp -r -u -n /srv/init/dvd/lib/modules/6.1.0-27-amd64/kernel/fs/* /srv/init/netboot/lib/modules/6.1.0-27-amd64/kernel/fs
cp -r -n -u /srv/init/dvd/lib/modules/6.1.0-27-amd64/kernel/drivers/* /srv/init/netboot/lib/modules/6.1.0-27-amd64/kernel/drivers/
#для проверки разницы файлов можно воспользоваться следующей командой
#diff -r /srv/init/dvd/lib/modules/6.1.0-27-amd64/kernel/drivers/ /srv/init/netboot/lib/modules/6.1.0-27-amd64/kernel/drivers/
#сейчас объединяем нашу подпись для репозитория с имеющейся на initrd на netboot
#объединение 2х файлов публичных ключей.  public.pgp (/srv/share/www/debian12/public.pgp) + debian-archive-keyring.gpg (находящегося в /srv/init/netboot/usr/share/keyrings/)
mv /srv/init/netboot/usr/share/keyrings/debian-archive-keyring.gpg /srv/init/netboot/usr/share/keyrings/debian-archive-keyring.gpg.orgn
cat /srv/share/www/debian12/public.pgp /srv/init/netboot/usr/share/keyrings/debian-archive-keyring.gpg.orgn > /srv/init/netboot/usr/share/keyrings/debian-archive-keyring.gpg
#дополнительно копируем получившийся файл в директорию для доступа по http 
cp /srv/init/netboot/usr/share/keyrings/debian-archive-keyring.gpg /srv/share/www/debian12 
#для запаковки initrd обратно  выполняем команды
#если необходимо для dvd запаковать, но это не надо.
#cd /srv/init/dvd/
#find .|cpio -H newc -o|gzip -9 > /srv/init/dvdz/initrd.gz - запаковка
#для netboot
cd /srv/init/netboot/
find .|cpio -H newc -o|gzip -9 > /srv/init/netbootz/initrd.gz
mv /srv/share/www/debian12/netboot/debian-installer/amd64/initrd.gz /srv/share/www/debian12/netboot/debian-installer/amd64/initrd.gz.orgn
cp /srv/init/netbootz/initrd.gz /srv/share/www/debian12/netboot/debian-installer/amd64/
#образ ядра копируем с DVD диска /mnt/cdrom/install.amd/vmlinuz
cp /mnt/cdrom/install.amd/vmlinuz /srv/share/www/debian12/netboot/debian-installer/amd64/
###############################################################################################
11) настройка репозитория.
#создание репозитория 
#создание каталогов, поиск и копирование пакетов
#каталог в который уже скопировали пакеты для репозитория /srv/share/www/debian12/cache
mkdir /srv/share/www/debian12/repo/conf
#!!!создаем файл с конфигурацией, SignWith берем из пункта создания подписи, в конце пункта 9 данной инструкции !!!
nano /srv/share/www/debian12/repo/conf/distributions
#текст ниже
Origin: Debian
Label: Debian
Suite: stable
Version: 12.8
Codename: bookworm
Architectures:amd64
Components:main contrib non-free-firmware
UDebComponents:main contrib non-free-firmware
Description: Apt repository for project DebianRepo
SignWith: 28E9A2D344FF7A86082A56616195B3635B1275C8
#
#Инициализация репозитория,reprepro создаст необходимые каталоги и файлы, потребуется пароль от ключа который создавали qwert
reprepro -b /srv/share/www/debian12/repo export bookworm
#выгрузка ключа для подписи репозитория
gpg --yes --armor -o /srv/share/www/debian12/repo/dists/bookworm/Release.gpg -sb /srv/share/www/debian12/repo/dists/bookworm/Release
#заполнить репозиторий пакетами из iso образа скопированными в /srv/share/www/debian12/cache/
reprepro --ask-passphrase -Vb /srv/share/www/debian12/repo includedeb  bookworm /srv/share/www/debian12/cache/*.deb
reprepro --ask-passphrase -Vb /srv/share/www/debian12/repo includeudeb bookworm /srv/share/www/debian12/cache/*.udeb
#создаем символьную ссылку
ln -s /srv/share/www/debian12/repo/dists/bookworm /srv/share/www/debian12/repo/dists/stable
#удалить ненужные каталоги, можно и не удалять, но каталог cache очень большой и будет дублировать пакеты которые скопированы в pool. Предлагаю оставить вдруг с первого раза не получится!
#rm -rf  /srv/share/www/debian12/repo/{cache,conf,db}
#Обновить файл с контрольными суммами
cd /srv/share/www/debian12/repo
find ! -name md5sum.txt ! -name gostsums.txt -follow -type f -print0 | xargs -0 md5sum > md5sum.txt ;
#объявляем владельцем каталога на www-data
chown -R www-data:www-data /srv/share/www
#для проверки подписи репозитория нашим ключем и объединенным нужно выполнить команды
gpg --no-default-keyring --keyring /srv/share/www/debian12/public.pgp --verify /srv/share/www/debian12/repo/dists/bookworm/Release.gpg /srv/share/www/debian12/repo/dists/bookworm/Release
gpg --no-default-keyring --keyring /srv/share/www/debian12/debian-archive-keyring.gpg --verify /srv/share/www/debian12/repo/dists/bookworm/Release.gpg /srv/share/www/debian12/repo/dists/bookworm/Release
#########################################################################################################
12) файл ответов и файл boot.ipxe
#сначала создаем boot.ipxe в /srv/share/ipxe/config
cd /srv/share/ipxe/config
nano /srv/share/ipxe/config/boot.ipxe
#текст ниже
#!ipxe
#dhcp
set menu-timeout 50000
set submenu-timeout ${menu-timeout}
set menu-default Load_disk
:start
menu iPXE boot menu
item --gap --                   -------------------------Load from hard disk------------------------------
item
item     Load_disk                      Load disk
item
item --gap --                   ------------------------- Operating systems ------------------------------
item
item	  debian12_8_bookworm_cli		      Debian 12.8 BookWorm CLI
item    debian12_8_bookworm_gnome       Debian 12.8 BookWorm GNOME
item
item --gap --                   ------------------------- Operating systems ------------------------------
item
item            shell                   Drop to iPXE shell
item            reboot                  Reboot
item
item --gap --                   --------------------- Restore Operating systems --------------------------
item
item   debian12_8_restore              Debian 12.8 Bookworm restore
item
item
item --key x    exit                    Exit iPXE and continue BIOS boot
choose --timeout ${menu-timeout} --default ${menu-default} selected || goto cancel
set menu-timeout 0
goto ${selected}

:Load_disk
sanboot --no-describe --drive 0x80
goto start

:cancel
echo You cancelled the menu, dropping you to a shell

:shell
echo Type 'exit' to get the back to the menu
shell
set menu-timeout 0
set submenu-timeout 0
goto start

:reboot
reboot

:exit
exit 1

:debian12_8_bookworm_cli
initrd http://192.168.2.4/debian12/netboot/debian-installer/amd64/initrd.gz
chain http://192.168.2.4/debian12/netboot/debian-installer/amd64/vmlinuz interface=auto auto=true locale=ru_RU language=ru country=RU hostname=debiantest domain=corp url=http://192.168.2.4/debian12/preseed-install-cli.cfg
goto start

:debian12_8_bookworm_gnome
initrd http://192.168.2.4/debian12/netboot/debian-installer/amd64/initrd.gz
chain http://192.168.2.4/debian12/netboot/debian-installer/amd64/vmlinuz interface=auto auto=true locale=ru_RU language=ru country=RU hostname=debiantest domain=corp url=http://192.168.2.4/debian12/preseed-install-gnome.cfg
goto start

:debian12_8_restore
initrd http://192.168.2.4/debian12/netboot/debian-installer/amd64/initrd.gz
chain http://192.168.2.4/debian12/netboot/debian-installer/amd64/linux interface=auto rescue/enable=true locale=ru_RU.UTF-8 language=ru country=RU
goto start

# конец файла boot.ipxe
#сохраняем  boot.ipxe
#################################
#генерация ssh ключа для вставки в файл ответов, если хотим подключаться к машинам без пароля, выполнить на том ПК с которого хотим получать доступ.
ssh-keygen -t ed25519
# далее берем сгенерированный открытый ключ из файла, добавляем ключ в файлы ответов в строку d-i preseed/late_command string
## далее файл ответов
# первый файл ответов для установки без GUI
cd /srv/share/www/debian12
nano /srv/share/www/debian12/preseed-install-cli.cfg
#текст ниже

#_preseed_V1
# Локализация
d-i debian-installer/language string ru
d-i debian-installer/country string RU
d-i debian-installer/locale string ru_RU.UTF-8
d-i debian-installer/keymap string ru

# Выбор клавиатуры.
d-i keyboard-configuration/xkb-keymap select ru
d-i keyboard-configuration/toggle select Alt+Shift
d-i keyboard-configuration/layoutcode string ru
d-i languagechooser/language-name-fb select Russian
d-i countrychooser/country-name select Russia
d-i console-setup/toggle string Alt+Shift
d-i console-setup/layoutcode string ru

### Настройка сети
# Выбор сетевого интерфеса
d-i netcfg/choose_interface select auto
# В том случае если мендленно работает DHCP
d-i netcfg/dhcp_timeout string 60

# Любые имена хостов и доменов, назначенные по протоколу dhcp, имеют приоритет над
# значения, установленные здесь. Однако установка значений по-прежнему предотвращает возникновение вопросов
# не будет отображаться, даже если значения поступают из dhcp.
d-i netcfg/get_hostname string debiantest
d-i netcfg/get_domain string corp

# Если вы хотите принудительно ввести имя хоста, независимо от того, что возвращает сервер DHCP
# или какова обратная запись DNS для IP, раскомментируйте
# и измените следующую строку.
d-i netcfg/hostname string debiantest

# Если для сети или другого оборудования требуется несвободная прошивка, вы можете
# настроить программу установки так, чтобы она всегда пыталась загрузить ее без запроса. Или
# установите значение false, чтобы отключить запрос.
d-i hw-detect/load_firmware boolean true

### Mirror settings
# Значение по умолчанию : http
#d-i mirror/protocol string ftp
d-i mirror/country string manual
d-i mirror/http/hostname string 192.168.2.4
d-i mirror/http/directory string /debian12/repo
d-i mirror/http/proxy string[/b]

# Пароль пользователя Root, виде открытого текста
d-i passwd/root-password password p@sswd
d-i passwd/root-password-again password p@sswd

# Для создания обычной учетной записи пользователя.
d-i passwd/user-fullname string userd
d-i passwd/username string userd
d-i passwd/user-password password 111111
d-i passwd/user-password-again password 111111

### Настройка часов и часового пояса
# Определяет, установлены ли аппаратные часы на UTC или нет.
d-i clock-setup/utc boolean true
# /usr/share/zoneinfo/ for valid values. (/usr/share/zoneinfo/Asia/Yekaterinburg)
d-i time/zone string Asia/Yekaterinburg

# Определяет, следует ли использовать протокол NTP для установки часов во время установки
d-i clock-setup/ntp boolean false
# Использовать NTP-сервер. Указать сервер NTP
#d-i clock-setup/ntp-server string ntp.example.com

# работа с дисками
d-i partman-auto/method string lvm
d-i partman-auto-lvm/guided_size string max
#. Это можно предварительно удалить...
d-i partman-lvm/device_remove_lvm boolean true
# То же самое относится и к ранее существовавшему программному RAID-массиву:
d-i partman-md/device_remove_md boolean true
#То же самое относится и к подтверждению записи разделов lvm.
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
# Имя группы томов lvm
d-i partman-auto-lvm/new_vg_name string debian
#Вы можете выбрать один из трех предопределенных рецептов разбиения на разделы:
# - atomic: все файлы в одном разделе
d-i partman-auto/choose_recipe select atomic
# Это приводит к автоматическому разделению partman без подтверждения, при условии, что
# вы указали ему, что делать, используя один из описанных выше методов.
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
#Принудительно загрузите UEFI ("Совместимость с BIOS" будет потеряна). По умолчанию: false. Закоментировать для поддержки BIOS Legacy
d-i partman-efi/non_efi_system boolean true
# Убедитесь, что таблица разделов является GPT - это необходимо для EFI
d-i partman-partitioning/choose_label select gpt
d-i partman-partitioning/default_label string gpt
# Если шифрование диска включено, пропустите предварительную очистку разделов.
#d-i partman-auto-crypto/erase_disks boolean false

### Apt setup
# Вы можете выбрать установку несвободной прошивки.
d-i apt-setup/non-free-firmware boolean true
# Вы можете выбрать установку несвободного или дополнительного программного обеспечения.
d-i apt-setup/contrib boolean true
# Доступны дополнительные репозитории, local[0-9], не добавлять текущий с которого будет установка иначе задублируются в /etc/apt/source.list
#d-i apt-setup/local0/repository string http://192.168.2.4/debian12/repo bookworm contrib main non-free-firmware
#d-i apt-setup/local0/comment string local server Enable deb-src lines
#d-i apt-setup/local0/source boolean true
# URL-адрес открытого ключа локального репозитория; вы должны предоставить ключ, иначе
# apt пожалуется на неподтвержденный репозиторий, и поэтому строка списка
# sources.будет оставлена закомментированной.
#d-i apt-setup/local0/key string http://192.168.2.4/debian12/debian-archive-keyring.gpg

### Выбор пакетов: standard (стандартные инструменты) desktop (графический рабочий стол)
tasksel tasksel/first multiselect standard, ssh-server

# Индивидуальные дополнительные пакеты для установки
d-i pkgsel/include string htop, curl

popularity-contest popularity-contest/participate boolean false

# Избегайте появления последнего сообщения о завершении установки.
d-i finish-install/reboot_in_progress note

# Эта команда выполняется непосредственно перед запуском разделителя. Это может быть
# полезно применять предварительную загрузку динамического разделителя, которая зависит от состояния
# дисков (которые могут быть невидимы при выполнении preseed/early_command).
#d-i partman/early_command \
#       string debconf-set partman-auto/disk "$(list-devices disk | head -n1)"
#добавим нашу подпись репозитория во вновь установленную систему
d-i partman/early_command string \
wget -O /tmp/apt_dak.gpg http://192.168.2.4/debian12/debian-archive-keyring.gpg; \
echo -e "cp /tmp/apt_dak.gpg /target/tmp/apt_dak.gpg\nchroot /target /bin/bash -c '/bin/cp /tmp/apt_dak.gpg /etc/apt/trusted.gpg.d/'" > /usr/lib/apt-setup/generators/001add-key ; \
chmod +x /usr/lib/apt-setup/generators/001add-key

# выполняется после установки, настройка SSH
d-i preseed/late_command string \
in-target sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config; \
in-target sed -i 's/#Port 22/Port 22/' /etc/ssh/sshd_config; \
in-target sh -c "echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILpW2MP5VcpU7TIrPT6Fudf0sywcdE9buf9S0joK3s92 root@testpxe' > /root/.ssh/authorized_keys";

#######################################################
#создаем еще один файл для установки с графическим интерфейсом gnome
nano /srv/share/www/debian12/preseed-install-gnome.cfg
#_preseed_V1

#Локализация
d-i debian-installer/language string ru
d-i debian-installer/country string RU
d-i debian-installer/locale string ru_RU.UTF-8
d-i debian-installer/keymap string ru

# Выбор клавиатуры.
d-i keyboard-configuration/xkb-keymap select ru
d-i keyboard-configuration/toggle select Alt+Shift
d-i keyboard-configuration/layoutcode string ru
d-i languagechooser/language-name-fb select Russian
d-i countrychooser/country-name select Russia
d-i console-setup/toggle string Alt+Shift
d-i console-setup/layoutcode string ru

### Настройка сети
#Выбор сетевого интерфеса
d-i netcfg/choose_interface select auto
# В том случае если мендленно работает DHCP
d-i netcfg/dhcp_timeout string 60

# Любые имена хостов и доменов, назначенные по протоколу dhcp, имеют приоритет над
# значения, установленные здесь. Однако установка значений по-прежнему предотвращает возникновение вопросов
# не будет отображаться, даже если значения поступают из dhcp.
d-i netcfg/get_hostname string debiantest
d-i netcfg/get_domain string corp

# Если вы хотите принудительно ввести имя хоста, независимо от того, что возвращает сервер DHCP
# или какова обратная запись DNS для IP, раскомментируйте
# и измените следующую строку.
d-i netcfg/hostname string debiantest

# Если для сети или другого оборудования требуется несвободная прошивка, вы можете
# настроить программу установки так, чтобы она всегда пыталась загрузить ее без запроса. Или
# установите значение false, чтобы отключить запрос.
d-i hw-detect/load_firmware boolean true

### Mirror settings
# Если вы выберете ftp, указывать строку "зеркало"/"страна" не нужно.
# Значение по умолчанию для зеркального протокола: http
#d-i mirror/protocol string ftp
d-i mirror/country string manual
d-i mirror/http/hostname string 192.168.2.4
d-i mirror/http/directory string /debian12/repo
d-i mirror/http/proxy string[/b]

# Пароль пользователя Root, виде открытого текста
d-i passwd/root-password password p@sswd
d-i passwd/root-password-again password p@sswd

# Для создания обычной учетной записи пользователя.
d-i passwd/user-fullname string userd
d-i passwd/username string userd
d-i passwd/user-password password 111111
d-i passwd/user-password-again password 111111

### Настройка часов и часового пояса
# Определяет, установлены ли аппаратные часы на UTC или нет.
d-i clock-setup/utc boolean true
# /usr/share/zoneinfo/ for valid values. (/usr/share/zoneinfo/Asia/Yekaterinburg)
d-i time/zone string Asia/Yekaterinburg

# Определяет, следует ли использовать протокол NTP для установки часов во время установки
d-i clock-setup/ntp boolean false
# Использовать NTP-сервер. Указать сервер NTP
#d-i clock-setup/ntp-server string ntp.example.com

# работа с дисками
d-i partman-auto/method string lvm
d-i partman-auto-lvm/guided_size string max
#. Это можно предварительно удалить...
d-i partman-lvm/device_remove_lvm boolean true
# То же самое относится и к ранее существовавшему программному RAID-массиву:
d-i partman-md/device_remove_md boolean true
#То же самое относится и к подтверждению записи разделов lvm.
d-i partman-lvm/confirm boolean true
d-i partman-lvm/confirm_nooverwrite boolean true
# Имя группы томов lvm
d-i partman-auto-lvm/new_vg_name string debian
#Вы можете выбрать один из трех предопределенных рецептов разбиения на разделы:
# - atomic: все файлы в одном разделе
d-i partman-auto/choose_recipe select atomic
# Это приводит к автоматическому разделению partman без подтверждения, при условии, что
# вы указали ему, что делать, используя один из описанных выше методов.
d-i partman-partitioning/confirm_write_new_label boolean true
d-i partman/choose_partition select finish
d-i partman/confirm boolean true
d-i partman/confirm_nooverwrite boolean true
#Принудительно загрузите UEFI ("Совместимость с BIOS" будет потеряна). По умолчанию: false.
d-i partman-efi/non_efi_system boolean true
# Убедитесь, что таблица разделов является GPT - это необходимо для EFI
d-i partman-partitioning/choose_label select gpt
d-i partman-partitioning/default_label string gpt
# Если шифрование диска включено, пропустите предварительную очистку разделов.
#d-i partman-auto-crypto/erase_disks boolean false

### Apt setup
# Вы можете выбрать установку несвободной прошивки.
d-i apt-setup/non-free-firmware boolean true
# Вы можете выбрать установку несвободного или дополнительного программного обеспечения.
d-i apt-setup/contrib boolean true
# Доступны дополнительные репозитории, local[0-9]
#d-i apt-setup/local0/repository string http://192.168.2.4/debian12/repo bookworm contrib main non-free-firmware
#d-i apt-setup/local0/comment string local server Enable deb-src lines
#d-i apt-setup/local0/source boolean true
# URL-адрес открытого ключа локального репозитория; вы должны предоставить ключ, иначе
# apt пожалуется на неподтвержденный репозиторий, и поэтому строка списка
# sources.будет оставлена закомментированной.
#d-i apt-setup/local0/key string http://192.168.2.4/debian12/debian-archive-keyring.gpg

### Выбор пакетов: standard (стандартные инструменты) desktop (графический рабочий стол)
tasksel tasksel/first multiselect standard, desktop, gnome-desktop, ssh-server

# Индивидуальные дополнительные пакеты для установки
d-i pkgsel/include string htop, curl

popularity-contest popularity-contest/participate boolean false

# Избегайте появления последнего сообщения о завершении установки.
d-i finish-install/reboot_in_progress note

# Эта команда выполняется непосредственно перед запуском разделителя. Это может быть
# полезно применять предварительную загрузку динамического разделителя, которая зависит от состояния
# дисков (которые могут быть невидимы при выполнении preseed/early_command).
#d-i partman/early_command \
#       string debconf-set partman-auto/disk "$(list-devices disk | head -n1)"
d-i partman/early_command string \
wget -O /tmp/apt_dak.gpg http://192.168.2.4/debian12/debian-archive-keyring.gpg; \
echo -e "cp /tmp/apt_dak.gpg /target/tmp/apt_dak.gpg\nchroot /target /bin/bash -c '/bin/cp /tmp/apt_dak.gpg /etc/apt/trusted.gpg.d/'" > /usr/lib/apt-setup/generators/001add-key ; \
chmod +x /usr/lib/apt-setup/generators/001add-key

# выполняется после установки
d-i preseed/late_command string \
in-target sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config; \
in-target sed -i 's/#Port 22/Port 22/' /etc/ssh/sshd_config; \
in-target sh -c "echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILpW2MP5VcpU7TIrPT6Fudf0sywcdE9buf9S0joK3s92 root@testpxe' > /root/.ssh/authorized_keys";

################################################################################################################
13) создаем виртуальную машину и тестируем
14) После установки закоментировать ненужные строки в файле списка репозиториев /etc/apt/sources.list, оставить только свой сетевой репозиторий или добавить по необходимости репы яндекса.

```








