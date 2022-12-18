# Ceph эксперименты

> Практическое обучение Ceph

Автор: Александр Тетеркин

Ceph — программно-определяемая распределенная горизонтально-масштабируемая система храненя данных, с открытым исходным кодом, обеспечивающая хранение данных на блочном, объектном и файловом уровнях.

Материал подготовлен на основе книги Mastering Ceph, 2nd edition.

## Подготовка

1. Установите VirtualBox: [VirtualBox Downloads](https://www.virtualbox.org/wiki/Downloads)
2. Установите Vagrant: [Install | Vagrant](https://developer.hashicorp.com/vagrant/downloads)

## Настройка Vagrant и Виртуального Окружения

В рамках настройки мы:

1. Создадим папку с проектом
2. Установим нужные плагины
3. Скачаем бокс (шаблон ВМ) с Ubuntu
4. Создадим конфигурационный файл Vagrant
5. Запустим 7 виртуальных машин для проверки
6. Зайдем на одну из машин для проверки
7. Удалим созданные для проверки работы кода машины

Выполните следующие команды для проверки:

```bash
# Создайте новую директорию для вашего проекта Vagrant
mkdir ceph-ansible

cd ceph-ansible

# Установите плагин
$ vagrant plugin install vagrant-hostmanager
Installing the 'vagrant-hostmanager' plugin. This can take a few minutes...
Fetching vagrant-hostmanager-1.8.9.gem
Installed the plugin 'vagrant-hostmanager (1.8.9)'!

# Добавьте новый бокс
$ vagrant box add bento/ubuntu-16.04        
==> box: Loading metadata for box 'bento/ubuntu-16.04'
    box: URL: https://vagrantcloud.com/bento/ubuntu-16.04
==> box: Adding box 'bento/ubuntu-16.04' (v202212.11.0) for provider: virtualbox
    box: Downloading: https://vagrantcloud.com/bento/boxes/ubuntu-16.04/versions/202212.11.0/providers/virtualbox.box
==> box: Successfully added box 'bento/ubuntu-16.04' (v202212.11.0) for 'virtualbox'!

# Создайте новый файл
vi Vagrantfile

# Содержимое файла
$ cat Vagrantfile
nodes = [ 
  { :hostname => 'ansible', :ip => '192.168.56.40', :box => 'xenial64' }, 
  { :hostname => 'mon1', :ip => '192.168.56.41', :box => 'xenial64' }, 
  { :hostname => 'mon2', :ip => '192.168.56.42', :box => 'xenial64' }, 
  { :hostname => 'mon3', :ip => '192.168.56.43', :box => 'xenial64' }, 
  { :hostname => 'osd1',  :ip => '192.168.56.51', :box => 'xenial64', :ram => 1024, :osd => 'yes' }, 
  { :hostname => 'osd2',  :ip => '192.168.56.52', :box => 'xenial64', :ram => 1024, :osd => 'yes' }, 
  { :hostname => 'osd3',  :ip => '192.168.56.53', :box => 'xenial64', :ram => 1024, :osd => 'yes' } 
] 
 
Vagrant.configure("2") do |config| 
  nodes.each do |node| 
    config.vm.define node[:hostname] do |nodeconfig| 
      nodeconfig.vm.box = "bento/ubuntu-16.04" 
      nodeconfig.vm.hostname = node[:hostname] 
      nodeconfig.vm.network :private_network, ip: node[:ip] 
 
      memory = node[:ram] ? node[:ram] : 512; 
      nodeconfig.vm.provider :virtualbox do |vb| 
        vb.customize [ 
          "modifyvm", :id, 
          "--memory", memory.to_s, 
        ] 
        if node[:osd] == "yes"         
          unless not File.exist?("disk_osd-#{node[:hostname]}")
            puts "Creating disk disk_osd-#{node[:hostname]}..."
            vb.customize [ "createhd", "--filename", "disk_osd-#{node[:hostname]}", "--size", "10000" ] 
          end 
          vb.customize [ "storageattach", :id, "--storagectl", "SATA Controller", "--port", 3, "--device", 0, "--type", "hdd", "--medium", "disk_osd-#{node[:hostname]}.vdi" ] 
        end 
      end 
    end 
    config.hostmanager.enabled = true 
    config.hostmanager.manage_guest = true 
  end 
end 

# Запустите машины
vagrant up

# Войдите в машину ansible
vagrant ssh ansible
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-210-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
vagrant@ansible:~$

# Нажмите exit для выхода
$ exit
logout
Connection to 127.0.0.1 closed.

# Удалите созданные 7 машин
vagrant destroy --force
```

## Установка и Настройка окружения Ansible

В рамках настройки окружения Ansible мы:

1. Запустим 3 узла из конфигурации, созданной ранее.
2. Добавим репозиторий Ansible и установим нужную версию
3. На уравляющем узле ansible сгенерируем ключ
4. Разошлем ключ на 2 узла mon1 и osd1 для беспарольного доступа
5. Проверим вход без пароля
6. Создадим конфигурационный файл Ansible
7. Создадим групповые переменные для групп mons и osds
8. Проверим связь Ansible с машиной
9. Создадим плэйбук и проверим работу с переменными

```bash
# Запустим три нужные нам узла:
vagrant up ansible mon1 osd1

# Перейдем в узел ansible
$ vagrant ssh ansible
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-210-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
vagrant@ansible:~$

# Добавим репозиторий Ansible нужной нам версии
$ sudo apt-add-repository ppa:ansible/ansible-2.6
 
 More info: https://launchpad.net/~ansible/+archive/ubuntu/ansible-2.6
Press [ENTER] to continue or ctrl-c to cancel adding it

gpg: keyring '/tmp/tmp155ge4df/secring.gpg' created
gpg: keyring '/tmp/tmp155ge4df/pubring.gpg' created
gpg: requesting key 7BB9C367 from hkp server keyserver.ubuntu.com
gpg: /tmp/tmp155ge4df/trustdb.gpg: trustdb created
gpg: key 7BB9C367: public key "Launchpad PPA for Ansible, Inc." imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
OK

# Обновим список пакетов и установим Ansible из добавленного нами репозитория
$ sudo apt-get update && sudo apt-get install ansible -y
...
Processing triggers for man-db (2.7.5-1) ...
Setting up python-markupsafe (0.23-2build2) ...
Setting up python-jinja2 (2.8-1ubuntu0.1) ...
Setting up python-yaml (3.11-3build1) ...
Setting up python-six (1.10.0-3) ...
Setting up python-ecdsa (0.13-2ubuntu0.16.04.1) ...
Setting up python-paramiko (1.16.0-1ubuntu0.2) ...
Setting up python-httplib2 (0.9.1+dfsg-1) ...
Setting up python-pkg-resources (20.7.0-1) ...
Setting up python-setuptools (20.7.0-1) ...
Setting up sshpass (1.05-1) ...
Setting up python-cffi-backend (1.5.2-1ubuntu1) ...
Setting up python-enum34 (1.1.2-1) ...
Setting up python-idna (2.0-3) ...
Setting up python-ipaddress (1.0.16-1) ...
Setting up python-pyasn1 (0.1.9-1) ...
Setting up python-cryptography (1.2.3-1ubuntu0.3) ...
Setting up ansible (2.6.20-1ppa~xenial) ...

# Генерация ключа без пароля
ssh-keygen

# Скопируем публичную часть ключа на узлы mon1 и osd1
# Пароль для доступа на mon1 под пользовталем vagrant — стандартный: vagrant

ssh-copy-id mon1

ssh-copy-id osd1

# Проверим доступ без пароля:
$ ssh mon1
Welcome to Ubuntu 16.04.7 LTS (GNU/Linux 4.4.0-210-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Sat Dec 17 19:05:59 2022 from 192.168.56.40
vagrant@mon1:~$

# Выйдем обратно на машину ansible
exit

# Создадим инвентарный файл Ansible
sudo vi /etc/ansible/hosts

# Посмотрим содержимое созданного файла:
$ sudo cat /etc/ansible/hosts
[mons]
mon1
mon2
mon3

[mgrs]
mon1

[osds]
osd1
osd2
osd3

[ceph:children]
mons
osds
mgrs

# Создадим директорию для переменных групп из инвентарного файла
sudo mkdir /etc/ansible/group_vars

# Создадим в данной директории два файла для групп mons и osds: 
sudo touch /etc/ansible/group_vars/{mons,osds}

# Добавим строчку в файл mons
sudo vi /etc/ansible/group_vars/mons

# Посмотрим содержимое файла
sudo cat /etc/ansible/group_vars/mons
a_variable: "foo"

# Добавим строчку в файл osds
sudo vi /etc/ansible/group_vars/osds

# Посмотрим содержимое файла
sudo cat /etc/ansible/group_vars/osds
a_variable: "bar"

# Проверим связь Ansible с машиной
$ ansible mon1 -m ping
mon1 | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}

# Запустим для проверки простую команду
$ ansible mon1 -a 'uname -r'
mon1 | SUCCESS | rc=0 >>
4.4.0-210-generic

# Для остановки машин можно использовать команду halt
$ vagrant halt       
==> osd3: VM not created. Moving on...
==> osd2: VM not created. Moving on...
==> osd1: Attempting graceful shutdown of VM...
==> mon3: VM not created. Moving on...
==> mon2: VM not created. Moving on...
==> mon1: Attempting graceful shutdown of VM...
==> ansible: Attempting graceful shutdown of VM...

# Запуск выполняется той же командой, что мы делали в самом начале:
# Запустим три нужные нам узла:
vagrant up ansible mon1 osd1

# Снова зайдем в машину ansible
vagrant ssh ansible

# Создадим для проверки плэйбук
sudo vi /etc/ansible/playbook.yml

# Содержимое файла
$ sudo cat /etc/ansible/playbook.yml 
- hosts: mon1 osd1
  tasks:
  - name: Echo Variables
    debug: msg="I am a {{ a_variable }}"

# Запустим плэйбук
# Обратите внимание, что плэйбук выводит на экран содержимое переменной a_variable, которую
# мы ранее определили в групповых переменных в group_vars.
$ ansible-playbook /etc/ansible/playbook.yml 

PLAY [mon1 osd1] *********************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************
ok: [mon1]
ok: [osd1]

TASK [Echo Variables] ****************************************************************************************************************
ok: [mon1] => {
    "msg": "I am a foo"
}
ok: [osd1] => {
    "msg": "I am a bar"
}

PLAY RECAP ***************************************************************************************************************************
mon1                       : ok=2    changed=0    unreachable=0    failed=0   
osd1                       : ok=2    changed=0    unreachable=0    failed=0

# Уберем созданные машины

exit

vagrant destroy --force
