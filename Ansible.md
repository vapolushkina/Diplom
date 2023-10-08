### playbook

```
---
- hosts: web
  become: true
  become_method: sudo
  become_user: root
  remote_user: admin
  roles:
   - role: nginx
   - role: node-exporter

- hosts: prom
  user: admin
  become: true
  become_method: sudo
  become_user: root
  roles:
    - role: prometheus

- hosts: grafana
  user: admin
  become: true
  become_method: sudo
  become_user: root
  roles:
    - role: grafana

- hosts: elastic
  user: admin
  become: true
  become_method: sudo
  become_user: root
  roles:
    - role: elastic

- hosts: kibana
  user: admin
  become: true
  become_method: sudo
  become_user: root
  roles:
    - role: kibana

- hosts: web
  become: true
  become_method: sudo
  become_user: root
  remote_user: admin
  roles:
   - role: filebeat

```
------

### nginx

1. `tasks`
```
- name: ensure nginx is at the latest version
  apt: name=nginx state=latest
  become: yes

- name: start nginx
  service:
     name: nginx
     state: started
  become: yes

- name: replace nginx.conf
  template:
     src=templates/nginx.conf
     dest=/etc/nginx/nginx.conf

- name: replace default
  template:
     src=templates/default
     dest=/etc/nginx/sites-enabled/default

- name: copy the content of the web site
  copy:
    src: /etc/ansible/roles/nginx/static/
    dest: /var/www/html

- name: restart nginx
  service:
     name: nginx
     state: restarted
  become: yes
```

2. `vars`
```
---
# vars file for roles/nginx
worker_processes: auto
worker_connections: 2048
client_max_body_size: 512M
```

3. `static`
```
---
<html>
  <head>
<meta charset="UTF-8">
<center><h1>Всего хорошего!</h1><center>
<img src="https://github.com/vapolushkina/vapolushkina/assets/121248099/5b6845f>
  </head>
  </html>
```
---------
### node-exporter
1. `tasks`
```
---
# tasks file for roles/node-exporter

- name: check if node exporter exist
  stat:
    path: "{{ node_exporter_bin }}"
  register: __check_node_exporter_present

- name: create node exporter user
  user:
    name: "{{ node_exporter_user }}"
    append: true
    shell: /usr/sbin/nologin
    system: true
    create_home: false

- name: create node exporter config dir
  file:
    path: "{{ node_exporter_dir_conf }}"
    state: directory
    owner: "{{ node_exporter_user }}"
    group: "{{ node_exporter_group }}"

- name: if node exporter exist get version
  shell: "cat /etc/systemd/system/node_exporter.service | grep Version | sed s/'.*Version '//g"
  when: __check_node_exporter_present.stat.exists == true
  changed_when: false
  register: __get_node_exporter_version

- name: download and unzip node exporter if not exist
  unarchive:
    src: "https://github.com/prometheus/node_exporter/releases/download/v{{ node_exporter_version }}/node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz"
    dest: /tmp/
    remote_src: yes
    validate_certs: no

- name: move the binary to the final destination
  copy:
    src: "/tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/node_exporter"
    dest: "{{ node_exporter_bin }}"
    owner: "{{ node_exporter_user }}"
    group: "{{ node_exporter_group }}"
    mode: 0755
    remote_src: yes
  when: __check_node_exporter_present.stat.exists == false or not __get_node_exporter_version.stdout == node_exporter_version

- name: clean
  file:
    path: /tmp/node_exporter-{{ node_exporter_version }}.linux-amd64/
    state: absent

- name: install service
  template:
    src: node.j2
    dest: /etc/systemd/system/node_exporter.service
    owner: root
    group: root
    mode: 0755

- name: service always started
  systemd:
    name: node_exporter
    state: started
    enabled: yes
```

2. `vars`
```
---
# vars file for roles/node-exporter

node_exporter_version: "1.6.1"
node_exporter_bin: /usr/local/bin/node_exporter
node_exporter_user: expo
node_exporter_group: "{{ node_exporter_user }}"
node_exporter_dir_conf: /etc/node_exporter

nginx_log_exporter : 1.9.2
```

3. `templates`
```
[Unit]
Description=Node Exporter Version {{ node_exporter_version }}
After=network-online.target
[Service]
User={{ node_exporter_user }}
Group={{ node_exporter_user }}
Type=simple
ExecStart={{ node_exporter_bin }}
[Install]
WantedBy=multi-user.target
```
--------
### prometheus
1. `tasks`
main.yml
```
---
# tasks file for roles/prometheus

- name: Install Prometheus
  include_tasks: tasks/prometheus.yml
```
prometheus.yml
```
- name: Create User prometheus
  user:
    name: prometheus
    create_home: no
    shell: /bin/false
- name: Create directories for prometheus
  file:
    path: "{{ item }}"
    state: directory
    owner: prometheus
    group: prometheus
  loop:
    - '/tmp/prometheus'
    - '/etc/prometheus'
    - '/var/lib/prometheus'
- name: Download And Unzipped Prometheus
  unarchive:
    src: https://github.com/prometheus/prometheus/releases/download/v{{ prometheus_version }}/prometheus-{{ prometheus_version }}.linux-amd64.tar.gz
    dest: /tmp/prometheus
    creates: /tmp/prometheus/prometheus-{{ prometheus_version }}.linux-amd64
    remote_src: yes
- name: Copy Bin Files From Unzipped to Prometheus
  copy: 
    src: /tmp/prometheus/prometheus-{{ prometheus_version }}.linux-amd64/{{ item }}
    dest: /usr/local/bin/
    remote_src: yes
    mode: preserve
    owner: prometheus
    group: prometheus
  loop: [ 'prometheus', 'promtool' ]
- name: Copy Conf Files From Unzipped to Prometheus
  copy: 
    src: /tmp/prometheus/prometheus-{{ prometheus_version }}.linux-amd64/{{ item }}
    dest: /etc/prometheus/
    remote_src: yes
    mode: preserve
    owner: prometheus
    group: prometheus
  loop: [ 'console_libraries', 'consoles', 'prometheus.yml' ]
- name: Add template
  template:
    src=templates/prometheus.yml
    dest=/etc/prometheus/prometheus.yml
- name: Create File for Prometheus Systemd
  template:
    src=templates/prometheus.service
    dest=/etc/systemd/system/
- name: Systemctl Prometheus Start
  systemd:
    name: prometheus
    state: started
    enabled: yes
```

2. `vars`
```
---
# vars file for roles/prometheus
prometheus_version : 2.23.0
```
3. `static`
prometheus.service
```
[Unit]
Description=Prometheus Service
After=network.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
```
prometheus.yml
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).
# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"
# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
   - job_name: 'node_exporter_clients'
     scrape_interval: 5s
     static_configs:
      - targets:
          - 192.168.10.10:9100
          - 192.168.11.10:9100
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
   - job_name: 'nginx_exporter'
     scrape_interval: 5s
     static_configs:
      - targets: 
          - 192.168.10.10:4040
          - 192.168.11.10:4040
```
--------
### grafana

1. `tasks`
```
---

# tasks file for roles/grafana
- name: Download grafana
  get_url:
    url: https://dl.grafana.com/oss/release/grafana_10.0.2_amd64.deb
    dest: /tmp/
- name: Install grafana
  command: "dpkg -i /tmp/grafana_10.0.2_amd64.deb"
- name: Systemctl grafana  Start
  systemd:
    name: grafana-server
    state: started
    enabled: yes
```
--------
### filebeat

1. `tasks`
```
---
# tasks file for roles/filebeat
- name: Create directories for filebeat
  file:
    path: "/tmp/filebeat"
    state: directory
- name: Download filebeat
  copy:
    src: "/etc/ansible/roles/kibana/static/filebeat-8.8.2-amd64.deb"
    dest: /tmp/filebeat
- name: Install filebeat
  apt:
    deb: "/tmp/filebeat/filebeat-8.8.2-amd64.deb"
- name: Copy template
  copy:
    src: "/etc/ansible/roles/filebeat/templates/filebeat.yml"
    dest: /etc/filebeat
- name: Copy module
  copy:
    src: "/etc/ansible/roles/filebeat/templates/nginx.yml"
    dest: /etc/filebeat/modules.d/
- name: Copy ca
  copy:
    src: "/etc/ansible/roles/kibana/static/http_ca.crt"
    dest: /etc/filebeat
```
--------
### kibana

1. `tasks`
```
---
# tasks file for roles/kibana
- name: Create directories for kibana
  file:
    path: "/tmp/kibana"
    state: directory
- name: Download kibana
  copy:
    src: "/etc/ansible/roles/kibana/static/kibana-8.8.2-amd64.deb"
    dest: /tmp/kibana
- name: Install kibana
  apt:
    deb: "/tmp/kibana/kibana-8.8.2-amd64.deb"
- name: Copy template
  copy:
    src: "/etc/ansible/roles/kibana/templates/kibana.yml"
    dest: /etc/kibana
- name: Create directories for ca
  file:
    path: "/etc/kibana/certs/"
    state: directory
- name: Copy ca
  copy:
    src: "/etc/ansible/roles/kibana/static/http_ca.crt"
    dest: /etc/kibana/certs/
```

