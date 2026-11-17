```yml
---

- name: update cache
  apt:
    update_cache: yes


- name: add GPG key docker
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present


- name: add repository docker
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present


- name: install packages
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
      - python3-pip
      - python3-setuptools
      - python3-docker
    state: present
    update_cache: yes


- name: run ane enable docker
  systemd:
    name: docker
    state: started
    enabled: yes


- name: create project folders
  file: 
    path: "{{app_dir}}/{{item}}"
    state: directory
    mode: '0755'
  loop:
    - "uploads/"
    - "nginx/"
    - "letsencrypt/"
    - "mysql/data/"


- name: copy template nginx
  template: 
    src: nginx.conf.j2
    dest: "{{app_dir}}/nginx/nginx.conf"
    mode: '0644'


- name: copy init mysql
  copy:
    src: init.sql
    dest: "{{app_dir}}/mysql/init.sql"
    mode: '0644'


- name: create docker network
  docker_network:
    name: app-network
    state: present
```