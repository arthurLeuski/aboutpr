Отчёт по настройке файлового сервера на базе Nginx, FTP и SFTP
1. Настройка HTTP‑файлового сервера на Nginx
### Создание директории
Создал директорию, в которой будут храниться файлы для скачивания:

bash
mkdir -p /var/www/fileexchange
В эту же папку добавил тестовые файлы.

### Конфигурация Nginx
Отредактировал файл /etc/nginx/sites-available/default, чтобы включить авто‑листинг файлов и корректную отдачу нужных расширений:

nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /var/www/fileexchange;

        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                try_files $uri $uri/ =404;
                autoindex_exact_size off;
                autoindex_localtime on;
                autoindex on;
        }

        location ~ \.(txt|pdf|zip|doc|docx|mp4|avi)$ {
                add_header Content-Disposition "attachment";
                try_files $uri =404;
        }
}
### Краткое описание ключевых параметров
autoindex on — включает отображение списка файлов вместо index.html

autoindex_localtime on — показывает локальное время изменения файлов

Content-Disposition "attachment" — браузер скачивает файл, а не открывает

try_files — отдаёт файл, иначе возвращает 404

После внесения изменений перезагрузил конфигурацию:

bash
sudo systemctl reload nginx
Теперь по IP можно зайти в браузере и увидеть список файлов.

2. Настройка FTP (vsftpd) для загрузки файлов
### Установка
bash
sudo apt install vsftpd -y
### Конфигурация
Отредактировал /etc/vsftpd.conf, чтобы можно было подключаться по FTP и загружать файлы в ту же директорию, что использует Nginx.

Основные параметры:
anonymous_enable=NO

write_enable=YES

chroot_local_user=YES

chroot_list_enable=YES

chroot_list_file=/etc/vsftpd.chroot_list

allow_writeable_chroot=YES

local_root=/var/www/fileexchange

xferlog_enable=YES

pasv_enable=YES

pasv_min_port=40000

pasv_max_port=50000

pasv_address=IP_ADDR

seccomp_sandbox=NO
### Что дают эти настройки

anonymous_enable=NO — отключает анонимный доступ

write_enable=YES — разрешает загрузку файлов

local_root=/var/www/fileexchange — сразу попадаю в нужную директорию

pasv_enable=YES — корректная работа FileZilla

chroot_local_user=YES — пользователь не может выйти выше своей директории

3. Настройка SFTP
Создал отдельный конфиг /etc/ssh/sshd_config.d/sftp.conf:

ssh
Subsystem sftp internal-sftp

Match User arthur
    ForceCommand internal-sftp
    ChrootDirectory /var/www
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no
### Смысл настроек
internal-sftp — используется встроенный SFTP
Match User arthur — правила применяются только к моему пользователю
ChrootDirectory /var/www — пользователь не может выйти выше этой директории
ForceCommand internal-sftp — доступ только к файлам, без возможности выполнять команды

4. Итог
В результате у меня получился рабочий мини‑файловый сервер:

Я могу подключаться по FTP или SFTP через FileZilla и загружать файлы.

Любой пользователь в моей сети может зайти по IP в браузере и скачать эти файлы.

Всё работает через одну общую директорию /var/www/fileexchange.

Доступ ограничен, базовая безопасность соблюдена.
