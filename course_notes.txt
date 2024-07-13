
ssh-keygen  -t ed25519 -C "ansible"
path-filename: /home/vacho/.ssh/ansible 
no passphrase

gets: 
./ssh/ansible.pub

ssh-copy-id -i ~/.ssh/ansible.pub 172.16.250.132
set a new passphrase for this

ssh -i ~/.ssh/ansible 172.16.250.132
(doesn't ask for a passphrase)

eval $(ssh-agent)
(cache the passphrase into 2362 pid)

ps aux | grep 2362
(show the background cache for 2362)

ssh-add 
(let add a passphrase)


alias ssha='eval $(ssh-agent) && ssh-add'
(creation of ssha command that save and add cache passphrase)

nano .bashrc
(thend add the alias ssha into this file to get this command permanently)

ansible all --key-file ~/.ssh/ansible -i inventory -m ping
(inventory is a list of ips, ping is a ansible module to ping a remote server)

ansible.cfg
[defaults]
inventory = inventory
private_key_file = ~/.ssh/ansible
remote_user = simone
(with remote_user = simone then can run: "ansible-playbook site.yml" as simone user )
$ ansible all -m ping
$ ansible all -m gather_facts
(show all keys that you can use for 'when' sentence)
$ ansible all -m apt -a update_cache=true --become --ask-become-pass
(--become is like a sudo in this case)
$ ansible all -m apt -a name=vim-nox --become --ask-become-pass
$ ansible all -m apt -a "name=snapd state=latest" --become --ask-become-pass
$ ansible all -m apt -a "upgrade=dist" --become --ask-become-pass
$ ansible all -m gather_facts --limit 172.16.250.248 | grep ansible_distribution

install_apache.yml
---

- hosts: all
  become: true
  tasks:
  
  - name: install apache and php packages
    apt:
	  name: 
	    - "{{ apache_package }}"
		- "{{ php_package }}"
	  state: latest
	  update_cache: yes

inventory
172.16.250.132 apache_package=apache2 php_package=libapache2-mod-php
172.16.250.133 apache_package=apache2 php_package=libapache2-mod-php
172.16.250.134 apache_package=apache2 php_package=libapache2-mod-php
172.16.250.248 apache_package=httpd php_package=php

$ ansible-playbook --ask-become-pass install_apache.yml

---------------------------------------
EVOLUTION
inventory

[web_servers]
172.16.250.132
172.16.250.248

[db_servers]
172.16.250.133

[file_servers]
172.16.250.134

[workstations]
172.16.250.135


site.yml
---

- host: all
  become: true
  pre_tasks:
  
  - name: install updates (CentOS)
    dnf:
	  update_only: yes
	  update_cache: yes
	when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    dnf:
	  upgrade: dist
	  update_cache: yes
	when: ansible_distribution == "Ubuntu"
	
- hosts: web_servers
  become: true
  tasks:
  
  - name: install apache and php for Ubuntu servers
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: install apache and php for CentOS servers
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"
	
- hosts: db_servers
  become: true
  tasks:
  
  - name: install mariadb package CentOS
    dnf:
      name: mariadb
      state: latest
    when: ansible_distribution == "CentOS"

  - name: install mariadb package Ubuntu
    apt:
      name: mariadb
      state: latest
    when: ansible_distribution == "Ubuntu"
	
- hosts: file_servers
  become: true
  tasks:
  
  - name: install samba package
    package:
	  name: samba
	  state: latest
	
$ ansible-playbook --ask-become-pass site.yml

-----------------------------------
Uninstall      

remove_apache.yml
---

- hosts: all
  become: true
  tasks:
  
  - name: update repository index
    apt:
	  update_cache: yes
	when: ansible_distribution = "Ununtu"
  
  - name: install apache package
    apt:
	  name: apache2
	  state: absent
	when: ansible_distribution = "Ununtu"
  
  - name: add php support for apache
    apt:
	  name: libapache2-mod-php
      state: absent
	when: ansible_distribution = "Ununtu"

$ ansible-playbook --ask-become-pass remove_apache.yml

-----------------------------------------------
Files

mkdir files
vim files/default_site.html

<html>
  <title>Web site test</title>
  <body>
    <p>Ansible is awesome!</p>
  </body>
</html>

site.yml

vim files/sudoer_simone
simone ALL=(ALL) NOPASSWD: ALL

sudoer_simone

---

- hosts: all
  become: true
  pre_tasks:
  
  - name: install updates (CentOS)
    dnf:
	  update_only: yes
	  update_cache: yes
	when: ansible_distribution == "CentOS"

  - name: install updates (Ubuntu)
    dnf:
	  upgrade: dist
	  update_cache: yes
	when: ansible_distribution == "Ubuntu"

- hosts: all
  become: true
  trasks:
  
  - name: create simone user
    tags: always
	user:
	  name: simone
	  groups: root
	  
  - name: add ssh key for simone
    tags: always
	authorized_key:
	  user: simone
	  key: "ssh-ed2q2 asdasdasdasdas..."
	  
  - name: add sudoers file for simone
    tags: always
    copy:
	  src: sudoer_simone
	  dest: /etc/sudoers.d/simone
	  owner: root
	  group: root
	  mode: 0440

- hosts: workstations
  become: true
  tasks:
  
  - name: install unzip
    package:
	  name: unzip

  - name: install terraform
    unarchive:
	  src: https://releases.hashiscorp.com/terraform/0.12.28/terraform_0.12.28_linux_amd64.zip
	  dest: /usr/local/bin
	  remote_src: yes 
	  owner: root
	  group: root
	  mode: 0755
	
- hosts: web_servers
  become: true
  tasks:
  
  - name: install apache and php for Ubuntu servers
    tags: apache, ubuntu, apache2
    apt:
      name:
        - apache2
        - libapache2-mod-php
      state: latest
    when: ansible_distribution == "Ubuntu"

  - name: install apache and php for CentOS servers
    tags: apache, centos, httpd
    dnf:
      name:
        - httpd
        - php
      state: latest
    when: ansible_distribution == "CentOS"
	
  - name: start httpd CentOS
    tags: apache, centos, httpd
	service:
	  name: httpd
	  state: started
	  enabled: yes
	when: ansible_distribution = "CentOS"

  - name: change e-mail address for admin
    tags: apache, centos, httpd
	lineinfile:
	  path: /etc/httpd/conf/httpd.conf
	  regexp: '^ServerAdmin'
	  line: ServerAdmin somebody@somewhere.net
	when: ansible_distribution = "CentOS"
	register: httpd

  - name: restart httpd CentOS
    tags: apache, centos, httpd
	service:
	  name: httpd
	  state: restarted
	when: httpd.changed
	  
	
  - name: copy default html file for site
    tags: apache, apache2, httpd
	copy:
	  src: default_site.html
	  dest: /var/www/html/index.html
	  owner: root
	  group: root
	  mode: 0664
	
- hosts: db_servers
  become: true
  tasks:
  
  - name: install mariadb package CentOS
    dnf:
      name: mariadb
      state: latest
    when: ansible_distribution == "CentOS"

  - name: install mariadb package Ubuntu
    apt:
      name: mariadb
      state: latest
    when: ansible_distribution == "Ubuntu"
	
- hosts: file_servers
  become: true
  tasks:
  
  - name: install samba package
    package:
	  name: samba
	  state: latest
	
$ ansible-playbook --ask-become-pass site.yml
