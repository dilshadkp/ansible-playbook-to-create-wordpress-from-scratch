# Fully automated Ansible Playbook to host a Wordpress site in an EC2

## Description

This is a fully automated Ansible Playbook to setup a Wordpress website from scratch. This playbook will launch an EC2 instance with a keypair and a secuirty group with required rules and then install the latest Wordpress website for you.


## Prerequisites

- Install Ansible in Ansible Master server. [Click here for Ansible Installation steps](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
- Attach IAM EC2 role to the Ansible master instance. (You can also use AWS access_key and secret_key to connect to AWS. In this case, you have to specify *access_key* and *secret_key* in your playbook)

 ## Features of this Playbook
 - While setting up the EC2 instance, the Playbook prompt you to enter ***AWS region, instance type, instance AMI id, your project name*** and ***name of SSH key***.
 - While installing Wordpress website, the Playbook prompt you to enter **domain name, new root password for mysql, name for your wordpress database** and ***database user name***
 - AWS IAM EC2 role is used instead of *secret_key* and *access_key* to connect to the AWS
 - This main.yml file actually contains 2 paybooks; one for launching the instance in AWS and one for installing and setting up Wordpress.



 ## Ansible Modules used in this Playbook
 - [vars_prompt](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)
 - [vars](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html)
 - [ec2_key](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_key_module.html)
 - [ec2_group](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_group_module.html)
 - [ec2](https://docs.ansible.com/ansible/latest/collections/amazon/aws/ec2_module.html)
 - [debug](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
 - [add_host](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/add_host_module.html)
 - [yum](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_module.html)
 - [copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)
 - [service](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)
 - [shell](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)
 - [template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
 - [file](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)
 - [mysql_user](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_user_module.html)
 - [mysql_db](https://docs.ansible.com/ansible/latest/collections/community/mysql/mysql_db_module.html)
 - [get_url](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html)
 - [unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)
  
- [with_items](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html) is used as a loop to delete files


 ## Tasks defined in the Playbook
 
>Below tasks will generate an SSH keypair and copy private key(pem) to the working directory of Ansible master

```python
    - name: "SSH keypair creation. Keyname is {{key_name}}"
      ec2_key:
        region: "{{region}}"
        name: "{{key_name}}"
        state: present
      register: "key_info"

    - name: "copy private key to local server"
      when: key_info.changed == true
      copy:
        content: "{{key_info.key.private_key}}"
        dest: "./{{key_name}}.pem"
        mode: "0400"
```
 >Below task will create a Security Group with ports 80,443,22 and 3306 opened

```python
    - name: "Create Security group"
      ec2_group:
        region: "{{region}}"
        name: "{{project_name}}-sg"
        description: "secuirty group for devops instance"
        tags:
          Name: "{{project_name}}-sg"          
        rules:
          - proto: tcp
            ports:
              - 80
              - 443
              - 22
              - 3306
            cidr_ip: 0.0.0.0/0
            cidr_ipv6: ::/0
            rule_desc: "allow all on ports 80, 443, 3306 and 22"
        rules_egress:
          - proto: -1
            ports: -1
            cidr_ip: 0.0.0.0/0
            cidr_ipv6: ::/0
      register: "sg_info"

```
>Below task will launch an EC2 instance of the submitted type using the submitted AMI id. The instance will use above created SSH key and Security Group. 

```python
    - name: "Create EC2"
      ec2:
        region: "{{region}}"
        key_name: "{{key_info.key.name}}"
        instance_type: "{{type}}"
        image: "{{ami_id}}"
        wait: yes
        group_id: "{{sg_info.group_id}}"
        instance_tags:
          Name: "{{project_name}}-webserver"
        count_tag:
          Name: "{{project_name}}-webserver"
        exact_count: 1
      register: ec2_info
```
>Below task will create an in-memory inventory file with the access details of above created instance. Second playbook will use this in-memory inventory file to access the server for setting up LAMPStack and Wordpress.

```python
    - name: "Create Dynamic inventory"
      add_host:
        groups: "webserver"
        hostname: "{{ ec2_info.tagged_instances[0].public_ip }}"
        ansible_host: "{{ ec2_info.tagged_instances[0].public_ip }}"
        ansible_user: "ec2-user"
        ansible_port: 22
        ansible_private_key_file: "{{key_name}}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
```
>Below tasks will install Apache and PHP 7.4 in the server. Then copy virtualhost configuration from template in the Master to apache configuration directory in the server.
>This also creates a phpinfo() page in the document root of the domain and lastly restart and enable apache service

```python
    - name: "Install apache"
      yum:
        name: 
          - httpd
        state: present

    - name: "Install PHP 7.4"
      shell: amazon-linux-extras install php7.4 -y

    - name: "copy virtualhost from template"
      template:
        src: vhost.conf.tmpl
        dest: /etc/httpd/conf.d/{{http_domain}}.conf

    - name: "create document root for website"
      file:
        path: /var/www/html/{{http_domain}}/
        state: directory
        owner: "{{http_user}}"
        group:  "{{http_group}}"

    - name: "create phpinfo page"
      copy:
        content: "<?php phpinfo(); ?>"
        dest: /var/www/html/{{http_domain}}/phpinfo.php
        owner: "{{http_user}}"
        group: "{{http_group}}"

    - name: "restart and enable apache service"
      service:
        name: httpd
        state: restarted
```
>Below tasks will install Mariadb-server in the server, set a password for mysql root user, remove test database and anonymous users.
Then it will create a database for wordpress installation and a db user with full privileges on the created database.

```python
    - name: "Install mariadb"
      yum:
        name: 
          - mariadb-server
          - MySQL-python
        state: present

    - name: "Restart and enable mariadb-server"
      service: 
        name: mariadb
        state: restarted
        enabled: true

    - name: "setting root password for mariadb"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        user: "root"
        password: "{{db_root_password}}"
        host_all: yes

    - name: "remove anonymous users"
      mysql_user:
        login_user: "root"
        login_password: "{{db_root_password}}"
        user: ""
        state: absent

    - name: "remove test database"
      mysql_db:
        login_user: "root"
        login_password: "{{db_root_password}}"
        name: "test"
        state: absent

    - name: "create wordpress database"
      mysql_db:
        login_user: "root"
        login_password: "{{db_root_password}}"
        name: "{{wp_db}}"
        state: present

    - name: "create wordpress db user"
      mysql_user:
        login_user: "root"
        login_password: "{{db_root_password}}"
        name: "{{wp_db_user}}"
        state: present
        password: "{{wp_db_user_password}}"
        priv: '{{wp_db}}.*:ALL'
```
>This part of the Playbook will download the latest version of Wordpress from the internet to the server, extract it and move it to the document root with proper ownership.
>Then it will copy the content from wp-config template in the master and paste into the document root of the website.
>Lastly, it will remove unwanted files from /tmp, restart the services and display domain name and Public IP after completing the whole setup.

```python
    - name: "Download wordpress from Internet"
      get_url:
        url: "{{download_link}}"
        dest: "/tmp/wordpress.tar.gz"
        remote_src: true

    - name: "extract tar file"
      unarchive:
        src: "/tmp/wordpress.tar.gz"
        dest: "/tmp/"
        remote_src: true

    - name: "copy wordpress files from /tmp to docroot"
      copy: 
        src: "/tmp/wordpress/"
        dest: /var/www/html/{{http_domain}}/
        owner: "{{http_user}}"
        group: "{{http_group}}"
        remote_src: true

    - name: "configuring wp-config.php from template"
      template:
        src: wp-config.php.tmpl
        dest: /var/www/html/{{http_domain}}/wp-config.php
        owner: "{{http_user}}"
        group: "{{http_group}}"

    - name: "clean-up unwanted files"
      file:
        path: "{{item}}"
        state: absent
      with_items:
        - "/tmp/wordpress"
        - "/tmp/wordpress.tar.gz"

    - name: "Final restart of apache"
      service:
        name: httpd
        state: restarted

    - name: "Print website details"
      debug:
        msg: 
        - "Your Wordpress website is ready. Please add below A RECORD in your DNS'"
        - "NAME=====> {{http_domain}}"
        - "TYPE=====> A"
        - "VALUE====>{{ ansible_ssh_host }}"
```
 ## Execution

 - Run a syntax check


 ```bash
ansible-playbook main.yml --syntax-check
```
 - Execute the Playbook


 ```bash
ansible-playbook main.yml
```
 ## Sample Output:

![](https://i.ibb.co/XZcj2Gv/11.png)
![](https://i.ibb.co/4fqvW5L/4.png)

x--------------------x---------------------x---------------------x---------------------x---------------------x---------------------x---------------------x---------------------x
### ⚙️ Connect with Me 

<p align="center">
<a href="mailto:dilshad.lalu@gmail.com"><img src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white"/></a>
<a href="https://www.linkedin.com/in/dilshadkp/"><img src="https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white"/></a> 
<a href="https://www.instagram.com/dilshad_a.k.a_lalu/"><img src="https://img.shields.io/badge/Instagram-E4405F?style=for-the-badge&logo=instagram&logoColor=white"/></a>
<a href="https://wa.me/%2B919567344212?text=This%20message%20from%20GitHub."><img src="https://img.shields.io/badge/WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white"/></a><br />
