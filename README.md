# ansible-homelab

## Basic config
This ansible project consists of the following files and folders:
- Inventory file
- Ansible.cfg
- Playbook
- Poles
- Group vars
- Vault

### Inventory
Ansible is used to managed multiple hosts, in order to do so, ansible is using a list called [inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#). The inventory file comes in two different formats: yaml and ini.

In this project the ini or hosts file will be used:

```shell
[homelab]
<ip_adress>

[homelab:vars]
ansible_user="{{ username }}"
ansible_port= # remove this if you don't intend to change ssh default port
ansible_ssh_private_key_file=
ansible_sudo_pass="{{ homelab_pass }}"
```
Basically we define an ip for the host we want to connect to, and we add some default variables.

The variable `"{{ username }}"` and `"{{ homelab_pass }}"` will be detailed in the group vars section.

### Ansible cfg
the ansible.cfg is nothing more than a configuration file.

```shell
[defaults]
inventory = hosts # location of the inventory file.
vault_password_file = <path_to_secret> # this will be required by ansible vault, you can remove it in you don't plan encrypt your secrets.
callbacks_enabled = profile_tasks # This will give us details such as the required to performed each tasks.

[colors] # simply the colors want to be displayed when running a playbook.
changed = purple
error = red
ok = bright green
skip = cyan
verbose = bright blue
debug = bright blue
```
### Playbook
The playbook is a blueprints that ansible uses to deploy and configure managed hosts. It consists of a play which is itself a list of task that will be executed one or more hosts listed in the inventory. To run a task ansible uses modules. In this project the playbook is called `homelab.yml`

```yaml
- name: Install homelab # the name of the play
  hosts: homelab # the
  tasks:
   - name: first task
     ansible.builtin.ping: # module used
```
To run a playbook you need to do `ansible-playbook homelab.yml`

### Roles
As you can already guess a playbook can get easily stuffed with a lot of tasks, so in order to keep our playbook readable we are going to use roles as a way to simplify our playbook.

We will do this:

```yaml
- hosts: homelab
  become: true
  roles:
    - traefik
```
Instead of:
```yaml
- hosts: homelab
  become: true
  tasks:
  - tags: traefik
    block:
      - name: Create traefik network
        docker_network:
          name: "{{ item.name }}"
          ipam_config:
            - subnet: "{{ item.subnet }}"
        loop:
          - { name: "{{ traefik_net }}", subnet: "{{ traefik_subnet }}"}
          - { name: "{{ socket_net }}", subnet: "{{ socket_subnet }}"}
  
      - name: Create working directory
        file:
          path: "{{ work_dir }}"
          state: directory
  
      - name: Create Crowdsec logs
        file:
          path: "{{ crowdsec_logs }}"
          state: directory
          owner: "{{ username }}"
          group: "{{ username }}"
  
      - name: Create acme file
        file:
          path: "{{ work_dir }}/acme.json"
          state: touch
          mode: 0600
  
      - name: Move files
        template:
          src: "{{ item.src }}"
          dest: "{{ work_dir }}"
        loop:
          - { src: "traefik.yml", dest: "{{ work_dir }}" }
          - { src: "config.yml", dest: "{{ work_dir }}" }
  
      - name: Create traefik
        docker_container:
          name: traefik
          image: "traefik:{{ version }}"
          state: started
          recreate: yes
          restart_policy: unless-stopped
          published_ports:
            - "80:80"
            - "443:443"
          volumes:
            - "{{ work_dir }}:/etc/traefik/"
            - "{{ crowdsec_logs }}:{{ crowdsec_logs }}"
          networks:
            - name: "{{ traefik_net }}"
              ipv4_address: "{{ traefik_ip }}"
            - name: "{{ socket_net }}"
          env:
            DOCKER_HOST: dockersocket
          labels:
            # Dashboard
            traefik.enable: "true"
            traefik.http.routers.traefik.rule: Host(`{{ subdomain }}`)
            traefik.http.routers.traefik.entryPoints: websecure
            traefik.http.routers.traefik.service: api@internal
            traefik.http.routers.traefik.tls.certresolver: letsencrypt
```

The different roles are stored in a folder called roles:

```shell
roles/
    common/               # this hierarchy represents a "role"
        tasks/            #
            main.yml      #  <-- tasks file can include smaller files if warranted
        handlers/         #
            main.yml      #  <-- handlers file
        templates/        #  <-- files for use with the template resource
            ntp.conf.j2   #  <------- templates end in .j2
        files/            #
            bar.txt       #  <-- files for use with the copy resource
            foo.sh        #  <-- script files for use with the script resource
        vars/             #
            main.yml      #  <-- variables associated with this role
        defaults/         #
            main.yml      #  <-- default lower priority variables for this role
        meta/             #
            main.yml      #  <-- role dependencies
        library/          # roles can also include custom modules
        module_utils/     # roles can also include custom module_utils
        lookup_plugins/   # or other types of plugins, like lookup in this case
```
The first folder under roles corresponds to the name of the roles.
more [info](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
### Group vars

The ansible group_vars is used to set variables for host and deploying plays against each host. The files under group_var are named after the host's name or all.yml depending on whether the variables need to be applied to that host or every host.

```shell
├── group_vars
   ├── all.yml
   └── homelab.yml
```
It is inside `homelab.yml` that we are going to store the variable you have seen in `hosts`

```yaml
username: test
traefik_net: traefik
domain: example.com
authelia_domain: "auth.{{ domain }}"
crowdsec_logs: /var/log/crowdsec/
homelab_pass: 0123456789
```
To use the variable just do `"{{ homelab_pass }}"`or whatever variable's name is.

### Vault

