## Описание файлов в директории
logFileFull.log - полный лог выполнения  
used_commands.txt - команды которые использовал  

Vagrant_folder - все что понадобится для поднятия VM и краткое описание файлов в ней  
├── Vagrantfile           - вагрант файл  
├── ansible.cfg  
├── playbooks  
│   ├── nginx.yml         - playbook  
│   └── templates  
│       └── nginx.conf.j2 - template file  
├── staging  
    └── hosts             - inventory file  

## Описание как запустить виртуальную машину (кратко)
Выполнить команду
```
vagrant up
```
Выполнить комманду, чтобы убедиться что nginx поднят и доступен
```
curl http://192.168.11.150:8080
```

# VagrantFile 
```
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :nginx => {
        :box_name => "centos/7",
        :ip_addr => '192.168.11.150'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "200"]
          end

          box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
          SHELL
          box.vm.provision "ansible", playbook: "playbooks/nginx.yml" #тут
      end
  end
end
```
box.vm.provision "ansible" - добавил строчку чтобы использовать ansible для провижинга  
box.vm.provision "shell" - это уже не нужно, но оставил если понадобится запустить ansible вручную

## Подробнее о ansible файлах
### ansible.cfg
```
[defaults]
inventory = staging/hosts   #путь к inventory
remote_user = vagrant		#пользователь для подключения
host_key_checking = False	#отвечает за проверку ключей при подключении по SSH
retry_files_enabled = False	#не создавать файлы retry
deprecation_warnings=false	#не выводить сообщения о неиспользуемых значениях
```

### nginx.yml
```
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080			#переменная с номером порта для nginx

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo #устнавливаем репозиторий epel-release
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo	#устанавливаем последнюю версию nginx
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Enable systemd 	NGINX			#включаем демон nginx
      systemd:
        name: nginx
        enabled : yes

    - name: NGINX | Create NGINX config file from template		#используем файл шаблона для формирования файла nginx.conf
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:							#если были изменения, то рестартуем демон nginx
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```

### nginx.conf.j2
```
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
```

### hosts
```
[web]
nginx ansible_host=127.0.0.1 ansible_port=2200 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```

📚Домашнее задание разработано для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)