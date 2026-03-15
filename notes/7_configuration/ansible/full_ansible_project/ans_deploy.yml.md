```yml
---
- name: Deploy rent ansible playbook 
  hosts: all
  become: yes

  roles:
    - role: first-setup  # first setup server (install docker for ansible)
      tags: [setup-server]

    - role: ssl-certificates  # get ssl certificates 
      tags: [ssl]
    
    - role: start-containers  # start project containers and healthcheck
      tags: [deploy]
```