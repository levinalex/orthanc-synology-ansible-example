# Orthanc Synology Setup

This is an example [ansible](http://docs.ansible.com/ansible/) playbook to set run the [fantastic Orthanc DICOM server](http://www.orthanc-server.com) on a Synology DiskStation with Docker.

### Prerequisites

  * ansible (I am using ansible version 2.3.0, installed with pip)
  * a Synology NAS that can run Docker (I am running this on a DS 1813+)

On your DiskStation, you need to create a user `ansible` with a strong password and enable SSH. (Terminal & SNMP in Control Panel).
Add that password and the IP address of your DiskStation to the `hosts` file.

After that, ansible should be able to connect to your NAS. You can verify with `ansible synology -m ping`.

- - -

Before running the playbook, you need to change the default passwords:

  * Edit `site.yml` to add your SSH public key and to set a strong password for your database.
  * Edit `synology-orthanc/templates/orthanc.json.j2` to configure Orthanc. You should change the default password for the web interface.


Running ansible-playbook with `ansible-playbook site.yml` should install and run Orthanc on your NAS.  You should be able to connect via `http://your_synology_ip:8042`.


## Overview of the playbook


#### Synology Preparation

The first thing this playbook does is enable SSH key based authentication on the Synology. (using the key given in `site.yml`)

```yaml
- name: Enable SSH Key Login for 'ansible' User
  become: yes
  authorized_key:
    user: ansible
    key: '{{ synology_ansible_ssh_key }}'
```

Then, because the `docker_container` [ansible module](https://docs.ansible.com/ansible/docker_container_module.html) needs the `docker-py` python package, the playbook installs pip:

```yaml
- name: check for pip
  shell: which pip
  changed_when: false
  failed_when: false
  register: pip_installed

- name: Download pip installer
  become: yes
  get_url:
    url: https://bootstrap.pypa.io/get-pip.py
    dest: /tmp/get-pip.py
  when: pip_installed.rc != 0

- name: Install pip
  become: yes
  command: python /tmp/get-pip.py
  when: pip_installed.rc != 0

- name: Install pip dependencies
  become: yes
  pip: name={{ item }}
  with_items:
    - docker-py
```

#### Create Directories and upload Orthanc configuration

The next steps create the necessary directories. The postgres data directory expects special permissions:

```yaml
- name: Create orthanc directories
  become: yes
  file:
    path: "{{ item }}"
    state: directory
    owner: ansible
    mode: 0755
  with_items:
    - '{{ orthanc_basedir }}'
    - '{{ orthanc_basedir }}/conf'
    - '{{ orthanc_basedir }}/storage'
    - '{{ orthanc_basedir }}/index'

- name: Create orthanc database directory
  become: yes
  file:
    path: '{{ orthanc_basedir }}/db'
    owner: 999
    group: 999
    state: directory
    mode: 0700
```

The next steps are for uploading the Orthanc configuration.

The orthanc configuration (`synology-orthanc/templates/orthanc.json.j2`) has the PostgreSQL plugin enabled and the postgres server host is set to the postgres container.


```json
{
  "PostgreSQL" : {
    "EnableIndex" : true,
    "EnableStorage" : false,
    "Host" : "{{ orthanc_name }}-postgres",
    "Port" : 5432,
    "Database" : "orthanc",
    "Username" : "orthanc",
    "Password" : "{{ orthanc_pg_password }}"
  }
}
```

If you have lua scripts, put them in `synology-orthanc/files/lua-scripts/` and add them to the configuration.

The configuration file is uploaded with these ansible steps:

```yaml
- name: Upload Orthanc configuration {{ orthanc_name }}
  template: src=orthanc.json.j2 dest={{ orthanc_basedir }}/conf/orthanc.json

- name: Upload orthanc scripts for {{ orthanc_name }}
  synchronize:
    src: orthanc-scripts/
    dest: '{{ orthanc_basedir }}/conf/scripts/'
    recursive: yes
    rsync_path: "rsync"
    delete: yes
  notify:
  -  Restart orthanc docker container {{ orthanc_name }}
```

#### Start Postgres and Orthanc Docker Images

Now postgres and Orthanc can be started:

```yaml
- name: Start docker container orthanc-postgres
  become: yes
  docker_container:
    name: 'orthanc-postgres'
    image: 'postgres:9.6.2'
    restart_policy: always
    env:
      TZ: UTC
      PGDATA: /var/lib/postgresql/data/pgdata
      POSTGRES_USER: orthanc
      POSTGRES_PASSWORD: 'password'
    volumes:
      - '{{ orthanc_basedir}}/db:/var/lib/postgresql/data/pgdata'
```


```yaml
- name: Orthanc docker container orthanc
  become: yes
  docker_container:
    name: 'orthanc'
    image: jodogne/orthanc-plugins:1.2.0
    command: '/etc/orthanc'
    restart_policy: always
    ports:
      - '4242:4242'
      - '8042:8042'
    links:
      - 'orthanc-postgres:orthanc-postgres'
    volumes:
      - '{{ orthanc_basedir }}/storage:/var/lib/orthanc/storage'
      - '{{ orthanc_basedir }}/index:/var/lib/orthanc/index'
      - '{{ orthanc_basedir }}/conf/orthanc.json:/etc/orthanc/orthanc.json:ro'
      - '{{ orthanc_basedir }}/conf/scripts:/etc/orthanc/scripts:ro'
```
