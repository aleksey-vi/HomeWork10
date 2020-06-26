# OTUS ДЗ 10 Управление пакетами. Дистрибьюция софта

## Домашнее задание

Размещаем свой RPM в своем репозитории
1) создать свой RPM (можно взять свое приложение, либо собрать к примеру апач с определенными опциями)
2) создать свой репо и разместить там свой RPM
реализовать это все либо в вагранте, либо развернуть у себя через nginx и дать ссылку на репо

### 1. создать свой RPM (можно взять свое приложение, либо собрать к примеру апач с определенными опциями)

Для начала нам необходимо установить нужный софт:

    sudo yum install -y -q epel-release > /dev/null 2>&1

Где **> /dev/null 2>&1** - означает, что вывод указанной команды будет отправлен в черную дыру, а не на экран))

    sudo yum install -y -q git rpm-build rpmdevtools gcc nano wget createrepo

    make automake yum-utils > /dev/null 2>&1

Установим необходимые для компиляции nginx devel-файлы

    sudo yum-builddep -y -q nginx > /dev/null 2>&1

Скачиваем rpm с исходниками nginx. Добавим репозиторий nginx (все как в мануале https://nginx.org/ru/linux_packages.html)

```bash
    cat <<'EOF1' | sudo tee /etc/yum.repos.d/nginx.repo
 [nginx]
 name=nginx repo
 baseurl=http://nginx.org/packages/mainline/centos/8/$basearch/
 gpgcheck=0
 enabled=1
 [nginx-source]
 name=nginx source repo
 baseurl=http://nginx.org/packages/mainline/centos/8/SRPMS/
 gpgcheck=0
 enabled=1
EOF1
```
Скачиваем исходники nginx, создаем в папке $HOME директории для сборки и распаковываем туда **nginx.src.rpm**

    yumdownloader --source nginx

    rpmdev-setuptree

    rpm -ivh /home/vagrant/nginx-* > /dev/null 2>&1

Для примера уберем поддержку ipv6

    sed -i '/--with-ipv6/d' ~/rpmbuild/SPECS/nginx.spec

Создаем rpm

    rpmbuild -bb ~/rpmbuild/SPECS/nginx.spec

На выходе мы получаем в директории **/home/vagrant/rpmbuild/RPMS/x86_64** файл **nginx-1.19.0-1.el8.ngx.src.rpm**

Теперь можно установить наш пакет и убедиться, что nginx работает:

    sudo yum localinstall -y /home/vagrant/rpmbuild/RPMS/x86_64/nginx-1.19.0-1.el7.ngx.x86_64.rpm

    systemctl start nginx

    systemctl status nginx

### 2. Создать свой репозиторий и разместить там ранее собранный RPM.

Теперь приступим к созданию своего репозитория. Директория для статики у NGINX по умолчанию **/usr/share/nginx/html**. Создадим там каталог repo:

    sudo mkdir /usr/share/nginx/html/repo

Копируем туда наш собранный RPM и RPM для установки репозитория Percona-Server:


    sudo cp /home/vagrant/rpmbuild/RPMS/x86_64/nginx-1.19.0-1.el7.ngx.x86_64.rpm /usr/share/nginx/html/repo/

    sudo wget http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm

Инициализируем репозиторий командой:

    sudo createrepo /usr/share/nginx/html/repo/

Для прозрачности настроим в NGINX доступ к листингу каталога: в location / в файле **/etc/nginx/conf.d/default.conf**. С помощью редактора nano добавим директиву ```autoindex on;```.

Проверяем синтаксис и перезапускаем NGINX:

    sudo nginx -t

    sudo nginx -s reload

Проверим с помощью curl:

    sudo curl -a http://localhost/repo/

Все готово для того, чтобы протестировать репозиторий. Добавим его в **/etc/yum.repos.d**:

    sudo nano /etc/yum.repos.d/otus.repo

Содержимое:

    [otus]
    name=otus-linux
    baseurl=http://localhost/repo
    gpgcheck=0
    enabled=1

Убедимся, что репозиторий подключился и посмотрим что в нем есть:

    yum repolist enabled | grep

    yum list | grep otus
Вывод должен быть таким:

    [root@otus x86_64]# yum list | grep otus
    percona-release.noarch                   0.1-6                         otus     
    [root@otus x86_64]# yum repolist enabled | grep otus
    otus                  otus-linux                                               2

Так как NGINX у нас уже стоит установим репозиторий percona-release:

    sudo yum install percona-release -y

Все прошло успешно. В случае если вам потребуется обновить репозиторий (а это делается при каждом добавлении файлов), снова то выполните команду **createrepo /usr/share/nginx/html/repo/**
