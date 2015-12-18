Title: Roles: 增强Playbooks功能
Date: 2015-12-17 11:01
Tags: ansible, roles

我喜欢Ansible其中一点原因是它能更好的向上及向下扩展。我不是指你所管理的主机数量，而是想实现自动化的工作的复杂性。
在Ansible里，role主要的机制就是把playbook拆分成多个文件，这样的好处就是简化编写playbook复杂度并且可复用。比如，database role会在一批机器部署成为数据库服务器。

## Role基本结构

Ansible role都有一个名称，如role “database”，数据库相关的配置都会放在roles/database，其中包含下列文件和目录：

    roles/database/tasks/main.yml
        #任务步骤Tasks
    roles/database/files/
        #存放上传到目标主机的文件
    roles/database/templates/
        #存放 Jinja2 模板 files
    roles/database/handlers/main.yml
        #处理程序类Handlers
    roles/database/vars/main.yml
        #不可覆盖的变量
    roles/database/defaults/main.yml
        #可以覆盖的缺省变量
    roles/database/meta/main.yml
        #角色的依赖信息

* 每个单独的文件都是可选的。如果你的角色没有任何handlers，就没有必要放一个空的handlers/main.yml文件。

---

Ansible 如何查找您定义的roles？

Ansible将在.playbooks同目录的roles目录中查找role，当然也会在系统默认的路径/etc/ansible/roles中查找role。

您可以修改ansible默认角色位置roles_path的值。
配置文件ansible.cfg：

    [defaults]
    roles_path = ~/ansible_roles
    #默认角色路径

您还可以通过设置环境变量ANSIBLE_ROLES_PATH覆盖掉roles_path的值。

---

## 例子: Database and Mezzanine Roles

我们定义2个角色"mezzanine"及"database"部署到一台主机，请看playbook例子：

mezzanine-single-host.yml

    - name: deploy mezzanine on vagrant
    hosts: web
    vars_files:
      - secrets.yml
    roles:
      - role: database
      database_name: "{{ mezzanine_proj_name }}"
      database_user: "{{ mezzanine_proj_name }}"

      - role: mezzanine
        live_hostname: 192.168.33.10.xip.io
        domains:
          - 192.168.33.10.xip.io
          - www.192.168.33.10.xip.io

当我们使用roles，playbook里有roles的键值，roles的值是列表，包含两个角色database 和 mezzanine。
例子中我们在当前的playbook定义了database_name 和 database_user的值，假设您已经也在vars/main.yml 或 defaults/main.yml定义了，值都会被当前的playbook的值覆盖。

如果role不需要定义变量，你可以简单地指定role名称，就像这样：

    roles:
      - database
      - mezzanine

例如，角色database部署在db组的机器、mezzanine部署在web组的机器。请注意，此playbook包含两个独立play。

    - name: deploy postgres on vagrant
      hosts: db
      vars_files:
        - secrets.yml
      roles:
        - role: database
        database_name: "{{ mezzanine_proj_name }}"
        database_user: "{{ mezzanine_proj_name }}"

    - name: deploy mezzanine on vagrant
      hosts: web
      vars_files:
        - secrets.yml
      roles:
        - role: mezzanine
        database_host: "{{ hostvars.db.ansible_eth1.ipv4.address }}"
        live_hostname: 192.168.33.10.xip.io
        domains:
          - 192.168.33.10.xip.io
          - www.192.168.33.10.xip.io

## Pre-Tasks 及 Post-Tasks

有时候，你想在角色部署之前或之后执行某些任务。比方说，你想你部署Mezzanine之前更新apt缓存，部署之后你想发送一条通知到slack。
Ansible允许您定义角色之前执行的任务放在pre_tasks列表，角色后执行的任务放在post_tasks列表，例如：

    - name: deploy mezzanine on vagrant
      hosts: web
      vars_files:
        - secrets.yml
      pre_tasks:
        - name: update the apt cache
          apt: update_cache=yes

      roles:
        - role: mezzanine
        database_host: "{{ hostvars.db.ansible_eth1.ipv4.address }}"
        live_hostname: 192.168.33.10.xip.io
        domains:
          - 192.168.33.10.xip.io
          - www.192.168.33.10.xip.io

      post_tasks:
        - name: notify Slack that the servers have been updated
          local_action: >
          slack
          domain=acme.slack.com
          token={{ slack_token }}
          msg="web server {{ inventory_hostname }} configured"

## 数据库部署角色 "Database" Role

下面我们演示一下"database" role部署：安装的Postgres并创建所需的数据库及数据库用户。

我们database角色包括以下文件：

* roles/database/tasks/main.yml
* roles/database/defaults/main.yml
* roles/database/handlers/main.yml
* roles/database/files/pg_hba.conf
* roles/database/files/postgresql.conf

此角色包括两个自定义的Postgres的配置文件。

postgresql.conf：

默认的Postgres仅接受来自本地主机的连接，所以修改默认的配置选项listen_addresses，这样Postgres将接受任何网络的连接。

pg_hba.conf：

Postgres认证使用的用户名和密码配置。

### 部署Postgres的Tasks文件 

roles/database/tasks/main.yml

    - name: install apt packages
      apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
      sudo: True
      with_items:
        - libpq-dev
        - postgresql
        - python-psycopg2
    - name: copy configuration file
      copy: >
        src=postgresql.conf dest=/etc/postgresql/9.3/main/postgresql.conf
        owner=postgres group=postgres mode=0644
      sudo: True
      notify: restart postgres
    - name: copy client authentication configuration file
      copy: >
        src=pg_hba.conf dest=/etc/postgresql/9.3/main/pg_hba.conf
        owner=postgres group=postgres mode=0640
      sudo: True
      notify: restart postgres
    - name: create a user
      postgresql_user:
        name: "{{ database_user }}"
        password: "{{ db_pass }}"
      sudo: True
      sudo_user: postgres
    - name: create the database
      postgresql_db:
        name: "{{ database_name }}"
        owner: "{{ database_user }}"
        encoding: UTF8
        lc_ctype: "{{ locale }}"
        lc_collate: "{{ locale }}"
        template: template0
      sudo: True
      sudo_user: postgres

### 部署Postgres的handlers文件 

roles/database/handlers/main.yml

    - name: restart postgres
      service: name=postgresql state=restarted
      sudo: True

### 部署Postgres的vars文件 

我们唯一需指定的变量是数据库端口。
roles/database/defaults/main.yml

    database_port: 5432

需要注意，以下变量没有在defaults中定义：

    database_name
    database_user
    db_pass
    locale

database_name、database_user在playbook中定义。db_pass是在secrets.yml中定义，在playbook中可以看到是vars_files中导入secrets.yml。locale是指所有部署的机器，所以可以在group_vars/all定义。

---

为什么有两个方法来定义角色中的变量？

当Ansible刚支持roles特性时，只有vars/main.yml可定义角色的变量，vars/main.yml变量优先级比playbooks中vars键中定义高，这意味着你不能覆盖变量，除非你明确地传参数（argument）给角色。
Ansible后来推出了默认角色变量概念defaults/main.yml。这类型的变量是在角色中定义，是低优先级，因此，如果在playbook中具有相同名称变量会将其覆盖。
如果你认为你可能要会角色变量值，使用defaults/main.yml变量。如果你不希望变量改变，就使用vars/main.yml变量。

---

## Mezzanine部署角色 "mezzanine" Role

mezzanine部署过程会部署nginx、supervisor。
具体的文件列表：

* roles/mezzanine/defaults/main.yml
* roles/mezzanine/handlers/main.yml
* roles/mezzanine/tasks/django.yml
* roles/mezzanine/tasks/main.yml
* roles/mezzanine/tasks/nginx.yml
* roles/mezzanine/templates/gunicorn.conf.py.j2
* roles/mezzanine/templates/local_settings.py.filters.j2
* roles/mezzanine/templates/local_settings.py.j2
* roles/mezzanine/templates/nginx.conf.j2
* roles/mezzanine/templates/supervisor.conf.j2
* roles/mezzanine/vars/main.yml

### 变量文件
如下面代码所示，我们定义mezzanine常量，由于ansible是没有命名空间，所以如果2个角色定义的变量名一样，可能会出现未知错误。下面例子的作法，变量名都是以role名作为前缀是比较好的作法。

/mezzanine/vars/main.yml

    # mezzanine vars 变量文件 
    mezzanine_user: "{{ ansible_ssh_user }}"
    mezzanine_venv_home: "{{ ansible_env.HOME }}"
    mezzanine_venv_path: "{{ mezzanine_venv_home }}/{{ mezzanine_proj_name }}"
    mezzanine_repo_url: git@github.com:lorin/mezzanine-example.git
    mezzanine_proj_dirname: project
    mezzanine_proj_path: "{{ mezzanine_venv_path }}/{{ mezzanine_proj_dirname }}"
    mezzanine_reqs_path: requirements.txt
    mezzanine_conf_path: /etc/nginx/conf
    mezzanine_python: "{{ mezzanine_venv_path }}/bin/python"
    mezzanine_manage: "{{ mezzanine_python }} {{ mezzanine_proj_path }}/manage.py"
    mezzanine_gunicorn_port: 8000

如下面的代码所示，我们定义mezzanine默认变量，这个变量就没有以role名作为前缀，因为我们会根据需要重新定义变量值。

roles/mezzanine/defaults/main.yml

    tls_enabled: True

### 任务文件

因为任务内容比较多，所以我们把task拆分到3个文件中。
mezzanine 最开始就是安装apt软件包，然后用include调用同目录的其它2个任务文件。

roles/mezzanine/tasks/main.yml

    - name: install apt packages
      apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
      sudo: True
      with_items:
        - git
        - libjpeg-dev
        - libpq-dev
        - memcached
        - nginx
        - python-dev
        - python-pip
        - python-psycopg2
        - python-setuptools
        - python-virtualenv
        - supervisor

      - include: django.yml
      - include: nginx.yml

roles/mezzanine/tasks/django.yml

    - name: check out the repository on the host
      git:
        repo: "{{ mezzanine_repo_url }}"
        dest: "{{ mezzanine_proj_path }}"
        accept_hostkey: yes
    - name: install required python packages
      pip: name={{ item }} virtualenv={{ mezzanine_venv_path }}
      with_items:
        - gunicorn
        - setproctitle
        - south
        - psycopg2
        - django-compressor
        - python-memcached
        - name: install requirements.txt
      pip: >
        requirements={{ mezzanine_proj_path }}/{{ mezzanine_reqs_path }}
        virtualenv={{ mezzanine_venv_path }}
    - name: generate the settings file
      template: src=local_settings.py.j2 dest={{ mezzanine_proj_path }}/local_settings.py
    - name: sync the database, apply migrations, collect static content
      django_manage:
        command: "{{ item }}"
        app_path: "{{ mezzanine_proj_path }}"
        virtualenv: "{{ mezzanine_venv_path }}"
      with_items:
        - syncdb
        - migrate
        - collectstatic
    - name: set the site id
      script: scripts/setsite.py
      environment:
        PATH: "{{ mezzanine_venv_path }}/bin"
        PROJECT_DIR: "{{ mezzanine_proj_path }}"
        WEBSITE_DOMAIN: "{{ live_hostname }}"
    - name: set the admin password
      script: scripts/setadmin.py
      environment:
        PATH: "{{ mezzanine_venv_path }}/bin"
        PROJECT_DIR: "{{ mezzanine_proj_path }}"
        ADMIN_PASSWORD: "{{ admin_pass }}"
    - name: set the gunicorn config file
      template: src=gunicorn.conf.py.j2 dest={{ mezzanine_proj_path }}/gunicorn.conf.py
    - name: set the supervisor config file
      template: src=supervisor.conf.j2 dest=/etc/supervisor/conf.d/mezzanine.conf
      sudo: True
      notify: restart supervisor
    - name: ensure config path exists
      file: path={{ mezzanine_conf_path }} state=directory
      sudo: True
      when: tls_enabled
    - name: install poll twitter cron job
      cron: >
        name="poll twitter" minute="*/5" user={{ mezzanine_user }}
        job="{{ mezzanine_manage }} poll_twitter"

/mezzanine/tasks/nginx.yml

    - name: set the nginx config file
      template: src=nginx.conf.j2 dest=/etc/nginx/sites-available/mezzanine.conf
      notify: restart nginx
      sudo: True
    - name: enable the nginx config file
      file:
        src: /etc/nginx/sites-available/mezzanine.conf
        dest: /etc/nginx/sites-enabled/mezzanine.conf
        state: link
      notify: restart nginx
      sudo: True
    - name: remove the default nginx config file
      file: path=/etc/nginx/sites-enabled/default state=absent
      notify: restart nginx
      sudo: True
    - name: create tls certificates
      command: >
        openssl req -new -x509 -nodes -out {{ mezzanine_proj_name }}.crt
        -keyout {{ mezzanine_proj_name }}.key -subj '/CN={{ domains[0] }}' -days 3650
        chdir={{ mezzanine_conf_path }}
        creates={{ mezzanine_conf_path }}/{{ mezzanine_proj_name }}.crt
      sudo: True
      when: tls_enabled
      notify: restart nginx

从上面的例子中我们可以看出，使用role的方案时，当使用copy时，ansible默认查找的目录rolename/files/，当使用template时，ansible默认查找的目录rolename/templates。类似下面的改变过程：

自己指定路径：

    - name: set the nginx config file
      template: src=templates/nginx.conf.j2 \
      dest=/etc/nginx/sites-available/mezzanine.conf

使用role方案（注意路径变化）：

    - name: set the nginx config file
      template: src=nginx.conf.j2 dest=/etc/nginx/sites-available/mezzanine.conf
      notify: restart nginx

### handlers 文件

roles/mezzanine/handlers/main.yml

    - name: restart supervisor
      supervisorctl: name=gunicorn_mezzanine state=restarted
      sudo: True
    - name: restart nginx
      service: name=nginx state=restarted
      sudo: True

## 使用ansible-galaxy创建role相关文件及目录

ansible-galaxy主要是用于下载Ansible社区分享的role部署方案。当然也可以用于初始化role的相关文件及目录。

    $ ansible-galaxy init -p playbooks/roles web --offline

-p是指定创建目录，如果不指定就会在当时目录创建roles。

    playbooks/roles/web/tasks/main.yml
    playbooks/roles/web/handlers/main.yml
    playbooks/roles/web/vars/main.yml
    playbooks/roles/web/defaults/main.yml
    playbooks/roles/web/meta/main.yml
    playbooks/roles/web/files/
    playbooks/roles/web/templates/
    playbooks/roles/web/README.md

## 角色依赖

假设我们有2个角色web和database，这2个角色部署的前提是先部署NTP1服务。哪我们可以分别在这2个角色安装前先部署NTP1服务，但这样会导致重复部署NTP1。当然我们也可以新建一个NTP1的角色，但我们必须记住在部署web和database前要先部署NTP1服务，虽然避免重复但很容易忘记或出错。所以我们需要有一个调度策略是部署web和database前需要检查NTP1服务是否存在。

Ansible使用角色依赖方案解决此类问题。当你定义一个角色，你可以定义它依赖于一个或多个角色，Ansible将会先执行它所依赖的角色。roles/web/meta/main.yml这个文件就是用于定义依赖关系。请看例子：

roles/web/meta/main.yml

    dependencies:
      - { role: ntp, ntp_server=ntp.ubuntu.com }

当然也可以指定依赖多个角色，假如django部署是依赖nginx和memcached先部署好的，我们可以如下定义：

roles/django/meta/main.yml

    dependencies:
      - { role: web }
      - { role: memcached }

## Ansible Galaxy

如果你需要部署一个开源系统到你的主机，有可能已经有人编写成ansible role了。虽然ansible编写部署脚本很容易，但有些系统部署还是蛮棘手的。

不管你是想重用别人编写好的角色部署脚本，还是说只是想看看别人是如何解决部署问题，都可以去Ansible Galaxy查找一下。Ansible Galaxy本身就是放在github上开源库。

[Ansible Galaxy](https://galaxy.ansible.com/)网站可以查找你需要部署的角色，同时也可以按类型及贡献者分类。

可以使用ansible-galaxy命令下载你所需要的角色。

比如我们要安装bennojoy编写的NTP server。

    $ ansible-galaxy install -p ./roles bennojoy.ntp

ansible-galaxy就会安装ntp到你的系统。

输出日志应该类似这样子：

    downloading role 'ntp', owned by bennojoy
    no version specified, installing master
    - downloading role from https://github.com/bennojoy/ntp/archive/master.tar.gz
    - extracting bennojoy.ntp to ./roles/bennojoy.ntp
    write_galaxy_install_info!
    bennojoy.ntp was installed successfully

ansible-galaxy就会安装ntp的文件到roles/bennojoy.ntp。

你也可查看通过ansible-galaxy安装的role：

    ansible-galaxy list

卸载命令：
    ansible-galaxy remove bennojoy.ntp


