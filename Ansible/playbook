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
