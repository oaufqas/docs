```yml
---

- name: check actual certificates
  stat:
    path: "{{ app_dir }}/letsencrypt/live/{{ domain }}/fullchain.pem"
  register: cert_file


- name: get SSL certificates (standalone)
  docker_container:
    name: certbot-tmp
    image: "{{ certbot_image }}"
    state: started
    cleanup: yes
    ports:
      - "80:80"
    detach: no
    restart_policy: no
    volumes:
      - "{{ app_dir }}/letsencrypt:/etc/letsencrypt"
    command: "certonly --standalone --email {{ admin_email }} --agree-tos --no-eff-email -d {{ domain }}"
  when: not cert_file.stat.exists
  register: certbot_result
```