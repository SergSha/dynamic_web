<h3>### DYNAMIC WEB ###</h3>

<p>развертывание веб приложения</p>

<h4>Описание домашнего задания</h4>

<ul>Варианты стенда:
<li>nginx + php-fpm (laravel/wordpress) + python (flask/django) + js(react/angular);</li>
<li>nginx + java (tomcat/jetty/netty) + go + ruby;</li>
<li>можно свои комбинации.</li>
</ul>

<ul>Реализации на выбор:
<li>на хостовой системе через конфиги в /etc;</li>
<li>деплой через docker-compose.</li>
</ul>

<p>Для усложнения можно попросить проекты у коллег с курсов по разработке</p>

<ul>К сдаче принимается:
<li>vagrant стэнд с проброшенными на локалхост портами</li>
<li>каждый порт на свой сайт</li>
<li>через nginx</li>
</ul>

<p>Формат сдачи ДЗ - vagrant + ansible</p>

<h4>Создание стенда Dynamic web</h4>

<p>Содержимое Vagrantfile:</p>

<pre>[user@localhost dynamic_web]$ <b>vi ./Vagrantfile</b></pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :master => {
    :box_name => "centos/7",
    :vm_name => "master",
    :ip => '192.168.50.10',
    :mem => '1048'
  },
  :replica => {
    :box_name => "centos/7",
    :vm_name => "replica",
    :ip => '192.168.50.11',
    :mem => '1048'
  },
  :backup => {
    :box_name => "centos/7",
    :vm_name => "backup",
    :ip => '192.168.50.12',
    :mem => '1048'
  }
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", boxconfig[:mem]]
      end
#      if boxconfig[:vm_name] == "backup"
#        box.vm.provision "ansible" do |ansible|
#          ansible.playbook = "ansible/playbook.yml"
#          ansible.inventory_path = "ansible/hosts"
#          ansible.become = true
#          ansible.verbose = "vvv"
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#        end
#      end
    end
  end
end</pre>

<p>Запустим виртуальные машины:</p>

<pre>[user@localhost dynamic_web]$ <b>vagrant up</b></pre>

<pre>[user@localhost dynamic_web]$ <b>vagrant status</b>
Current machine states:

master                    running (virtualbox)
replica                   running (virtualbox)
backup                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost dynamic_web]$</pre>

<h4>Физическая репликация</h4>

<h4>Сервер master</h4>