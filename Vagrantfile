# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :otus => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        :ip_addr => '192.168.11.101',
  },

}

Vagrant.configure("2") do |config|

	config.vm.box_version = "1804.02"
		MACHINES.each do |boxname, boxconfig|
		        config.vbguest.no_install = true

			config.vm.define boxname do |box|
			box.vm.box = boxconfig[:box_name]
			box.vm.host_name = boxname.to_s
			#box.vm.network "forwarded_port", guest: 3260, host: 3260+offset
			box.vm.network "private_network", ip: boxconfig[:ip_addr]
			box.vm.provider :virtualbox do |vb|
				vb.customize ["modifyvm", :id, "--memory", "1024"]
               		end
			box.vm.provision "shell", inline: <<-SHELL
				mkdir -p ~root/.ssh
				cp ~vagrant/.ssh/auth* ~root/.ssh
			SHELL
			
		
box.vm.provision "shell", privileged: false, inline: <<-SHELL
        echo -e "\n\nЗадание: собрать собственный rpm пакет и разместить его в собственном репозитории"
        echo -e "\nСборка кастомного nginx..."
        echo -e "Устанавливаем необходимый софт..."
        sudo yum install -y -q epel-release > /dev/null 2>&1
        sudo yum install -y -q git rpm-build rpmdevtools gcc make automake yum-utils wget createrepo > /dev/null 2>&1
        echo -e "\nУстановим необходимые для компиляции nginx devel-файлы"
        sudo yum-builddep -y -q nginx > /dev/null 2>&1
        echo -e "\nСкачиваем rpm с исходниками nginx..."
        echo -e "\nДобавим репозиторий nginx..."
cat <<'EOF1' | sudo tee /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/mainline/centos/7/$basearch/
gpgcheck=0
enabled=1
[nginx-source]
name=nginx source repo
baseurl=http://nginx.org/packages/mainline/centos/7/SRPMS/
gpgcheck=0
enabled=1
EOF1
        yumdownloader --source nginx > /dev/null 2>&1
        rpmdev-setuptree > /dev/null 2>&1
        rpm -ivh /home/vagrant/nginx-* > /dev/null 2>&1
        echo "Для примера уберем поддержку ipv6"
        sed -i '/--with-ipv6/d' ~/rpmbuild/SPECS/nginx.spec
        echo "Создаем rpm..."
        rpmbuild -bb ~/rpmbuild/SPECS/nginx.spec > /dev/null 2>&1
        echo "Готово"
	echo "Теперь можно установить наш пакет и убедиться что nginx работает"
	sudo yum localinstall -y /home/vagrant/rpmbuild/RPMS/x86_64/nginx-1.19.0-1.el7.ngx.x86_64.rpm > /dev/null 2>&1
        sudo systemctl start nginx
        sudo systemctl status nginx
	echo -e "\n\nЗадание: собрать собственный rpm пакет и разместить его в собственном репозитории"
	echo -e "\n\nТеперь приступим к созданию своего репозитория. В директории для статики создадим свой каталог repo"
        sudo mkdir /usr/share/nginx/html/repo
	echo -e "\nКопируем туда наш собранный RPM и RPM для установки репозитория Percona-Server"
	sudo cp /home/vagrant/rpmbuild/RPMS/x86_64/nginx-1.19.0-1.el7.ngx.x86_64.rpm /usr/share/nginx/html/repo/
	sudo wget http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm > /dev/null 2>&1
	echo -e "\nИнициализируем репозиторий"	
        sudo createrepo /usr/share/nginx/html/repo/ > /dev/null 2>&1
	echo -e "\nПерезапустим nginx"
	sudo nginx -s reload
        echo -e "\nСоздаем файл repo..."
cat <<'EOF' | sudo tee /etc/yum.repos.d/otus.repo
[otus]
name=otus-linux
baseurl=http://localhost/repo
enabled=1
gpgcheck=0
EOF
	echo "Так как NGINX у нас уже стоит установим репозиторий percona-release:"
        sudo yum install percona-release -y

	SHELL
end

end
end
	
