Задание:

Создать плейбук и роль Nginx Vhost.\
Роль должна устанавливать на целевой хост nginx и активировать vhost c заданным именем.\
Для vhost использовать j2 шаблон индексного файла и конфига.\
Родительский плейбук должен вызывать роль и список vhost передавать через переменную уровня полей в цикле.\
Результат выложить на Github, в качестве ответа приложить ссылку на репо.



Инвентарный файл
```yaml
---
all:
  children:
    webservers:
      hosts:
        devops-01:
        devops-02:
...
```


Структура роли
```yaml
├── inventory.yml
├── playbook.yml
├── README.md
└── roles
    └── nginx_vhosts
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   └── main.yml
        └── templates
            ├── index.html.j2
            └── site.conf.j2
```


Playbook
```yaml
---
- name: Install_nginx_with_virtual_hosts
  become: true
  hosts: webservers
  become_method: ansible.builtin.sudo
  gather_facts: false

  vars:
    sites:
      - "mehmat.ru"
      - "fizfak.ru"
      - "etis.com"
  roles:
    - nginx_vhosts
...
```



Роль  nginx-vhosts
roles/nginx_vhosts/tasks/main.yml
```yaml
---
- name: Install_nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Copy_nginx_config_from_template
  ansible.builtin.template:
    src: 'site.conf.j2'
    dest: '/etc/nginx/conf.d/site-{{ item }}.conf'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ sites }}'
  notify: Reload_nginx

- name: Create_sites_folders
  ansible.builtin.file:
    state: directory
    path: '/var/www/{{ item }}/'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ sites }}'

- name: Copy_index_from_template
  ansible.builtin.template:
    src: 'index.html.j2'
    dest: '/var/www/{{ item }}/index.html'
    owner: root
    group: root
    mode: '0644'
  loop: '{{ sites }}'
...
```

Шаблон конфига виртуального хоста
roles/nginx_vhosts/templates/site.conf.j2

```html
server {
        listen 80;
       
        root /var/www/{{ item }}/;
        index index.html;
	
	server_name {{ item }};
	
	location / {
		try_files $uri $uri/ =404;
	}
}
```

Шаблона стартовой страницы (index) виртуального сайта (vhost)
roles/nginx_vhosts/templates/index.html.j2

```html
<HTML>
  <BODY>
    Hello! Site: {{ item }}
  </BODY>
</HTML>
```

Задача для handler
roles/nginx_vhosts/handlers/main.yml
```yaml
---
- name: Reload_nginx
  ansible.builtin.systemd:
    name: nginx
    state: reloaded
...
```


Проверка линтером на ошибки
```bash
yamllint playbook.yml roles/nginx-vhosts/tasks/main.yml roles/nginx-vhosts/handlers/main.yml
```
Тестовый прогон playbook-а
```bash
ansible-playbook playbook.yml -i inventory.yml --check
```
Боевой прогон playbook-а
```bash
ansible-playbook playbook.yml -i inventory.yml --check
```

Проверка через curl
```bash
curl devops-01 -H "Host: mehmat.ru"
```
```html
<HTML>
  <BODY>
    Hello! Site: mehmat.ru
  </BODY>
</HTML>
```
 ```bash
 curl devops-01 -H "Host: fizfak.ru"
 ```
 ```html
<HTML>
  <BODY>
    Hello! Site: fizfak.ru
  </BODY>
</HTML>
```
 ```bash
 curl devops-01 -H "Host: etis.com"
 ```
```html 
<HTML>
  <BODY>
    Hello! Site: etis.com
  </BODY>
</HTML>
```

```bash
curl devops-02 -H "Host: mehmat.ru"
```
```html
<HTML>
  <BODY>
    Hello! Site: mehmat.ru
  </BODY>
</HTML>
```
 ```bash
 curl devops-02 -H "Host: fizfak.ru"
 ```
 ```html
<HTML>
  <BODY>
    Hello! Site: fizfak.ru
  </BODY>
</HTML>
```
 ```bash
 curl devops-02 -H "Host: etis.com"
 ```
```html 
<HTML>
  <BODY>
    Hello! Site: etis.com
  </BODY>
</HTML>
```


Создается workflows Ansible lint 
```bash
mkdir .github
cd .github/
cd workflows/
vim lint.yml
```

```yaml
---
name: ansible-lint
on:
  push:
    branches: ["master"]
jobs:
  build:
    name: Ansible Lint
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
      - name: Run ansible-lint
        uses: ansible/ansible-lint@main
...
```
Добовляем в репозитоий, делаем commit и push-им в Github
```bash
git add .
git commit -m "add worklflows ansible-lint"
git push
```
