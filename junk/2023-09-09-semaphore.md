---
title: Semaphore - An open-source Ansible GUI
image: semaphore.png
img_path: /images/
date: 2023-09-09
categories: [homelabbing]
tags: [ansible,semaphore,automation]
pin: false
comments: true
---

Ansible, is an agentless python-based automation tool. It works with `playbooks` and `inventory` lists in order to execute state changes based on system information it pulls before execution called `facts`. Ansible is a configuration management tool and can both keep your machines in a certain state, or make changes on the fly. Semaphore takes Ansible one step further by creating an open source web GUI around Ansible and Github to make executing playbooks and managing inventory lists simple and easy.

## Introduction to Ansible

Let's learn a bit about Ansible. Ansible uses `playbooks` which are yml based files. These playbooks are meant to work with Ansible's many modules in order to send commands to remote servers. Ansible does this via SSH, so it does not require an agent. First Ansible makes the SSH connection, then it pulls data on the state of the machine. Using this state data, Ansible then executes your playbook to make the state of the machine match the state of the playbook (Such as "all apt packages are updated"). Your inventory is where you store the servers you want to act upon.

## Ansible Installation

Ansible can be setup in a PUSH or PULL configuration, but this guide focuses on the push method using an Ubuntu server. In this method a server is setup as a controller in order to carry out commands against your hosts.

Prerequisites: Ubuntu Server, Python -- 

```bash
sudo apt install software-properties-common # needed for using add-apt-repository command if not installed
sudo add-apt-repository --yes --update ppa:ansible/ansible # add latest ansible repository and update
sudo apt install sshpass # for sshing using a password automation
ansible --version # check to make sure it works
```

## Simple Ansible directories

While Ansible can have a complicated file structure for managing roles and groups, here we will just create the necessary basic folders. Create a structure like this:

```text
Ansible/
├── inventory.ini (this can be a yaml or ini file. I prefer ini)
├── ansible.cfg
|   
├── Playbooks/ (directory contains playbooks)
│   ├── facts.yml
│   ├── update_hosts.yml
│   └── reboot_hosts.yml
```


## Inventory example

```text
[ubuntu_hosts]
subdomain.example.com
othersubdomain.example.com

[ubuntu_hosts:vars]
ansible_user=ansible_man
ansible_ssh_private_key_file=/home/ansible_man/.ssh/ansible_key.pem

[rhel_hosts]
rhelexample.example.com
192.168.10.10

[rhel_hosts:vars]
ansible_user=rhel_user
ansible_ssh_private_key_file=/home/rhel_user/.ssh/rhel_key.pem
```

Here we are simply specifying two host types, ubuntu_hosts and rhel_hosts. Servers can be listed in FQDN format or in IP address format. Next we define variables to be used when ansible is acting upon these servers using the `server:vars` format. We use this here to specify the user to login to the server with as well as the private key to use. Environment variables don't have to be used in an inventory file though! In fact, ansible allows for specifying these types of variables within the inventory, within playbooks, one off commands, or even in an environment variables file. How you manage your variables depends on design preference and use case. 

Next let's create a Playbook file. 

# First Playbook

> Keep in mind YML files are based off tabbing structure. If the tabs aren't correct, it will not work. 
{: .prompt-warning }

```yaml
#facts.yml example
- hosts: all
  tasks:
    - name: print facts # This playbook simply prints useful facts about hosts
      debug:
        msg:
          - "The hostname is: {{ inventory_hostname }}"
          - "IPv4 address: {{ ansible_default_ipv4.address }}"
```

In the example above, we use ansible to print out the hostname and IPv4 address of every host in our inventory list. Let's run this playbook against our inventory file. The following command points to our playbook and inventory files to run Ansible.

```bash
ansible-playbook -K ~/ansible/playbooks/facts.yml -i ~/ansible/inventory
```

That's great and all, but let's save some time and set some defaults in ansible. We do this and much more by creating a simple ansible.cfg file in the base directory of /ansible.

```text
# ansible.cfg

[defaults]
inventory = /home/username/ansible/inventory # specify where to default look for inventory file
remote_user = username # default username to login with
private_key_file = /home/username/.ssh/keyname.pem # default key to use
interpreter_python = auto_silent # default Python interpreter detection. Automatic is the
								 # default as of latest version and I add silent to get rid 
                                 # of a warning you get with Rocky linux.
```

Finally, If you're using a `passphrase` with your SSH key it's important to load this into an ssh-agent on your ansible controller or it won't work properly. This caches your SSH passphrase so ansible can flip easily through hosts. Add as many keys as you use. 

```bash
eval "$(ssh-agent -s)" #run ssh agent
ssh-add ~/.ssh/yourkey.pem #add you key - fill out passphrase
```

>One problem with SSH agents is by default they won't kill themselves and they will continue to store your passphrase and key location. Sometimes if you're not careful you can end up with multiple sessions of them running and not even know it. For this reason and many others Ansible typically accesses machines using a well guarded passphraseless key or by username and password authentication for a user specifically setup for Ansible in your environment.
{: .prompt-info }

Now that we have this setup properly we can just specify the playbook to run ansible:

```bash
ansible-playbook -K ~/ansible/playbooks/facts.yml
```

## Scratching the surface

We've learned a lot. We know how to create a simple ansible directory including an inventory list with variables, and execute playbooks quick and efficiently using configuration files. While these are the basics of Ansible there are some huge concepts missing here. Below are some topics you may want to explore to up your ansible game.

### Tags

There's a lot you can do with tags. Tags in their simplest form are added within playbooks so you can call a specific task or a group of tasks when necessary. An example of tag usage is found below:

```yaml
- hosts: "*"
  become: yes

  tasks:

  - name: Install the servers
    yum:
      name:
      - httpd
      - memcached
      state: present
      tags:
      - packages
      - webservers
  
  - name: Add database
    yum:
      name:
       - mariadb-server
      state: present
      tags:
      - database

  - name: Copy over index file
    copy:
      src: ~/ansible/files/index.html
      dest: /var/www/html/index.html
      owner: ubuntu
      group: ubuntu
      mode: '0764'
      tags: 
      - webservers
      - movethefile
    when: inventory_hostname == "192.168.10.10" or inventory_hostname == "192.168.10.11"
```

Firstly you might notice in this example we use a when statement. You can run certain tasks in Ansible based off conditionals and that is what I did here. 

Also in this example we are using the three tags packages, memcached, and database. The playbook itself could be used to fully setup a web server with an apache handler, maria database back end, and copy the main webpage over to the website directory. However, the advantage of tags is we can add as many as we want and then call them accordingly. For example, if you wanted to install everything on all hosts you would just call:

```bash
ansible-playbook -K ~/ansible/playbooks/playbook.yml
```

However, if you wanted to only setup for a web server with no database you could call the same playbook with tag webservers which matches everything but the database task:

```bash
ansible-playbook --tags webservers -K ~/ansible/playbooks/playbook.yml
```

What if we had a custom html page for a server we were spinning up and didn't want the file move portion but DID need the database? Very easily we add the appropriate tags in our call:

```bash
ansible-playbook --tags "packages,database" -K ~/ansible/playbooks/playbook.yml
```

We now understand the amazing usefulness of tags and the organization they bring to Ansible. This just scratches the surface and tags can be way more complicated including auto assigning them and special tags.

### Includes

At a high level includes are used to break up playbooks into task lists which can be imported into other playbooks for organization and ease of use. Yet another way Ansible gives you tools to design your architecture however you'd like. Here we execute a debug task printing the message "task1" and then executing some tasks from another file called sometasks.yml, and then finally printing "task3".

```yaml
- hosts: all
  tasks:
    - debug:
        msg: task1

    - name: Include task list in play
      include_tasks:
        file: sometasks.yaml

    - debug:
        msg: task3
```
### Handlers

In order to execute tasks only when certain conditions are met Ansible uses `Handlers`. Handlers are included at the end of a playbook and are only executed if they are called using a command called a `notify`. In this example below you can see during the playbook we ask the services to be restarted by notifying them. They will not execute this until the end of the playbook, and will not execute more than once regardless of how many times they are called.

```yaml
tasks:
- name: Template configuration file
  ansible.builtin.template:
    src: template.j2
    dest: /etc/foo.conf
  notify:
    - Restart apache
    - Restart memcached

handlers:
  - name: Restart memcached
    ansible.builtin.service:
      name: memcached
      state: restarted

  - name: Restart apache
    ansible.builtin.service:
      name: apache
      state: restarted
```

### Roles

Roles can be complicated, but pretty much the standard of use for modern Ansible usage. Think of roles as a way of packaging all the necessary files needed to perform a certain function all under one umbrella. The umbrella in this case, is a certain file structure. 

Ansible has a built in way to create this structure:

```bash
ansible-galaxy init /etc/ansible/roles/apache --offline
```

Here is a directory tree example of what Ansible creates:

```text
/etc/ansible/roles/apache/
|-- README.md
|-- defaults
|   `-- main.yml
|-- files
|-- handlers
|   `-- main.yml
|-- meta
|   `-- main.yml
|-- tasks
|   `-- main.yml
|-- templates
|-- tests
|   |-- inventory
|   `-- test.yml
`-- vars
    `-- main.yml
```

Now that we have a directory structure setup we can fill in the files. Tasks can be pasted into the tasks main.yml or they can be put in their own files and called within main.yml for organization. Handlers, templates, variables, and everything we need is contained within the role. Now we just simply call the role within another playbook.

```yaml
---
 - hosts: node2
   roles:
   - apache
```

### Templates

Templates are typically used to edit and create configuration files dynamically. Let's say you wanted to use Ansible to generate an HTML page which had a dynamic title and post description? It's best to see a practical example for this by looking at both the file that is templated, and the playbook which inserts variables into the template.

The template file (template.html)

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{{ page_title }}</title>
  <meta name="description" content="Created with Ansible">
</head>
<body>
    <h1>{{ page_title }}</h1>
    <p>{{ page_description }}</p>
</body>
</html>
```

You'll notice this has two variables waiting to be filled in within the template, `page_title` and `page_description`.


The playbook:

```text
---
- hosts: all
  become: yes
  vars:
    page_title: My Landing Page
    page_description: This is my landing page description.
  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: latest

    - name: Apply Page Template
      template:
        src: files/landing-page.html.j2
        dest: /var/www/html/index.nginx-debian.html

    - name: Allow all access to tcp port 80
      ufw:
        rule: allow
        port: '80'
        proto: tcp
```

In this playbook the two variables we wanted to fill in when we executed the playbook are defined. We then execute the playbook, the variables will be filled into the appropriate HTML variable spots, and we will have a complete file deployed.

![generated html template](landing-page.png){: w="840" h="400" }

## Semaphore - A GUI









