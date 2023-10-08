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

