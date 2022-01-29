## Using Ansible to set up a Debian Server

**Sven Haardiek, 2016-03-03**

In my last blog entry [Setup a Debian
Server](https://blog.haardiek.org/setup-a-debian-server.html) I described a
basic setup for a server running [Debian Jessie](https://www.debian.org). Now I
want to talk about setting up the same server but using
[Ansible](https://www.ansible.com). This is more or less an extension of the
other blog entry, but could be used by any Debian installation with ssh access
enabled for root using a password.


## Ansible

Ansible has multiple use cases and one of them is as a [configuration
management tool](https://www.ansible.com/configuration-management).

Instead of installing some kind of client on the target system the
configuration is done via ssh connections which is a great advantage because
you will leave a much smaller footprint on the target system.

We use [Playbooks](https://docs.ansible.com/ansible/playbooks.html) to define
the configuration state of the target system. The [Ansible
documentation](https://docs.ansible.com/ansible/playbooks.html) describes
Playbooks as following:

```
Playbooks are Ansible’s configuration, deployment, and orchestration
language. They can describe a policy you want your remote systems to enforce,
or a set of steps in a general IT process.

[...]

At a basic level, playbooks can be used to manage configurations of and
deployments to remote machines. At a more advanced level, they can sequence
multi-tier rollouts involving rolling updates, and can delegate actions to
other hosts, interacting with monitoring servers and load balancers along
the way.
```

Is is also important to understand that we do not define which commands should
run on the target system to get the state we want to have but that we define
the state of the target system with those Playbooks. Ansible does the remaining
work and run the commands to configure the system but only if it is necessary.
Otherwise nothing is done. So we can execute those Playbooks
multiple times and nothing change.

These Playbooks are written in [Yaml](http://yaml.org) and are therefor really
easy to read. So you can use them even as documentation.

Ansible is able to run [ad-hoc
commands](https://docs.ansible.com/ansible/intro_adhoc.html) on multiple target
system at once (This could be used for updates for example) or outsource
definitions in so called
[roles](https://docs.ansible.com/ansible/playbooks_roles.html), so that they
can be used in multiple Playbooks. There are a lot of other things Ansible is
able to to and you can set up complex configuration definition. If you want to
do that, look at [documentation](https://docs.ansible.com/).

So lets talk about the different files we have to define the state of the
Debian Server we want to set up.

```bash
$ ls
debianserver.yml  hosts  prebook.yml
```

First we want to have a look at the `hosts` file.

## Inventory file `hosts`

It is possible to configure multiple system with Ansible. To coordinate the
different systems and groups of multiple systems you want to administrate with
Ansible there exists a Inventory file. The most common Ansible installation
ship a global Inventory file under `/etc/ansible/hosts`, but since this file is
only editable with higher privileges, I prefer using a local Inventory file
`hosts`. Lets say we want to configure `server.com`. So `hosts` looks as
following

```
[server]
server.com
```

In our Playbooks we use the group name `server`. Therefor it would be very easy
to add multiple systems and run the Playbook against them. For example if we
want to configure `server.com` and let us say `192.133.46.13`, we simply add
the IP to `hosts`

```
[server]
server.com
192.133.46.13
```

It is also possible to add explicit ports, a range of IPs, and a lot of other
stuff. Simply look at [Inventory file
documentation](https://docs.ansible.com/ansible/intro_inventory.html).


## Playbooks

Since we want to configure the OpenSSH Server via a ssh connection, we have a
special use case here for Ansible. Because of that I decided to split the
configuration into two Playbooks. The first one `prebook.yml` changes the root
password and add a configuration user. The second one `debianserver.yml` uses
the configuration user and configures the remaining stuff like the OpenSSH
server and the firewall. Without separation we could not run the Playbook
multiple times because we would disable root access via ssh and therefor could
not use the connection the next time. With a separation we can run
`prebook.yml` as often as we want to and if we are happy, we could run
`debianserver.yml`. Also we can later change or add stuff to `debianserver.yml`
and run it again.

Now I describe the two Playbooks. Some shortsome short explanations are added
but if you want to understand it exactly look at the documentation of Ansible.

### Playbook `prebook.yml`

The next file we want to look at is `prebook.yml`. This is our first Playbook.

```yaml

---

# Prerequisite:
#   ssh access with root and password
#   ssh authentication key on local machine

- hosts: server


  vars_prompt:

    - name: root_password
      prompt: Enter new root password
      private: yes
      encrypt: sha512_crypt
      confirm: yes
      salt_size: 7

    - name: user_password
      prompt: Enter new user password
      private: yes
      encrypt: sha512_crypt
      confirm: yes
      salt_size: 7

    - name: ssh_key
      prompt: Enter filename of public ssh key
      default: "~/.ssh/id_rsa.pub"
      private: no


  remote_user: root


  tasks:

  - name: apply root password
    user:
      name: root
      password: "{{ root_password }}"

  - name: ensure sudo is installed
    apt:
      name: sudo

  - name: ensure user is present
    user:
      name: user
      append: yes
      groups: sudo
      password: "{{ user_password }}"

  - name: ensure user is accessable with ssh key
    authorized_key:
      user: user
      {% raw  %}key: "{{ lookup('file', ssh_key) }}"{% endraw %}
```

Here is a short explanation of the different parts:

```yaml
- hosts: server
```

defines the remote machines this Playbook is used for.

```yaml
  vars_prompt:

    - name: root_password
      prompt: Enter new root password
      private: yes
      encrypt: sha512_crypt
      confirm: yes
      salt_size: 7

    - name: user_password
      prompt: Enter new user password
      private: yes
      encrypt: sha512_crypt
      confirm: yes
      salt_size: 7

    - name: ssh_key
      prompt: Enter filename of public ssh key
      default: "~/.ssh/id_rsa.pub"
      private: no
```

is used to prompt some options to the user who runs the Playbook. The user can
set the new password for root and for the user used later for configuration.
Only a hashed value of the password is stored and not the password itself.
Furthermore he can set the path to the public ssh key used for later
identification.  See
[Prompts](https://docs.ansible.com/ansible/playbooks_prompts.html) for more
informations.

```yaml
  remote_user: root
```

defines the user used on the target system. For this user the ssh connection is
set and also under this user the commands run on the target system.

```yaml
  tasks:

  - name: apply root password
    user:
      name: root
      password: "{{ root_password }}"

  - name: ensure sudo is installed
    apt:
      name: sudo

  - name: ensure user is present
    user:
      name: user
      append: yes
      groups: sudo
      password: "{{ user_password }}"

  - name: ensure user is accessable with ssh key
    authorized_key:
      user: user
      {% raw  %}key: "{{ lookup('file', ssh_key) }}"{% endraw %}
```

This part defines the tasks running on the target system. The different tasks
are modules defined in Ansible itself, see
[Modules](https://docs.ansible.com/ansible/modules.html).

For example the module
[user](https://docs.ansible.com/ansible/user_module.html) is able to manage
user accounts. You only define the state you want to have on the target system.
Here you want to have a user called *user* present. This user should also be in
the group *sudo* and have set the password from the previous prompt. After
running the Playbook this is the state you have. Independent from what was
before. This is the great advantage about configuration management tools.

To look up a module used here, see the [Module
Index](https://docs.ansible.com/ansible/modules_by_category.html)

Simply run the following command to apply the Playbook.

```bash
ansible-playbook --inventory hosts prebook.yml --ask-pass
```

`--ask-pass` tells Ansible to try connecting with ssh using a password and not
a ssh key. As I said you can run this Playbook multiple times.


### Playbook `debianserver.yml`

Now lets have a look at the remaining Playbook.


```yaml
---

# Prerequisits: SSH access for user with password

- hosts: server


  remote_user: user


  tasks:

  - name: Do not allow root access via ssh
    sudo: yes
    lineinfile:
      line: PermitRootLogin no
      dest: /etc/ssh/sshd_config
      regexp: ^PermitRootLogin
    notify: restart ssh

  - name: Do not allow ssh access via password
    sudo: yes
    lineinfile:
      line: PasswordAuthentication no
      dest: /etc/ssh/sshd_config
      regexp: ^PasswordAuthentication
    notify: restart ssh

  - name: Delete ssh host keys
    sudo: yes
    shell: "rm /etc/ssh/ssh_host_* && dpkg-reconfigure openssh-server"
    notify: restart ssh
    when: renew_ssh is defined and renew_ssh == "yes"

  - name: Make sure uft is installed
    sudo: yes
    apt:
      name: ufw

  - name: Configure firewall
    sudo: yes
    ufw:
      policy: deny
      rule: allow
      name: OpenSSH
      state: enabled


  handlers:
    - name: restart ssh
      sudo: yes
      service:
        name: ssh
        state: restarted
```

Again here some short explanations. Things from `prebook.yml` are not repeated.

```yaml
    sudo: yes
```

Tasks and Handler run as another user (here as superuser).

```yaml
    notify: restart ssh
```

If the Tasks really changes something the Handler with the name `restart ssh`
runs at the end of the Playbook. This is most commonly used the restart or
reload services, but it is not limited to that.

```yaml
    when: renew_ssh is defined and renew_ssh == "yes"
```

This Tasks is skipped if the statement is `False`.

```yaml
  handlers:
    - name: restart ssh
      sudo: yes
      service:
        name: ssh
        state: restarted
```

In the Handlers section the different Handlers are defined.

As you maybe saw in the Playbook above the variable `renew_ssh` is used but
never defined. In Ansible you can define extra variables over the command line.
So if you do not set `renew_ssh` the replacement of the ssh keys is skipped.
This is reasonable since you do not want to change the ssh host keys every
time. So if you execute the Playbook with the following command

```bash
ansible-playbook --inventory hosts debianserver.yml --ask-sudo-pass
```

the ssh host keys are not replaced. But if you use

```bash
ansible-playbook -i hosts --ask-sudo-pass --extra-vars="renew_ssh=yes" debianserver.yml
```

you will get new ssh host keys.

`--ask-sudo-pass` prompt for the password of the user used for configuration.

## Final Words

After running both Playbooks against your server you should have the same state
than the one described in [Setup a Debian
Server](https://blog.haardiek.org/setup-a-debian-server.html).

Notice! You never manually logged in your system. So you leave it in a very
clean state. Every change is documented in the Playbooks.

You can also try new changes on a test server and later apply it to your
production server. But by using Ansible and automate the process it is much
harder to make errors during the transfer.

Also if you server dies for some reason for example because the hardware
breaks. You can set up a new one with the exact same state. And that very
quick. Of course you data will be lost!

In conclusion I would like to encourage you look deeper into Ansible. I hope I
was able to show you some advantages against the manual setup of machines.
