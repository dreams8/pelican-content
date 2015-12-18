Title: Ansible节点配置文件(Inventory)
Date: 2015-12-15 14:01
Tags: ansible, inventory

[1]: http://docs.ansible.com/ansible/intro_inventory.html "Inventory"
[2]: ansible/intro_dynamic_inventory.html "动态获取节点配置文件"
[3]: ansible/YAMLSyntax.html "YAMLSyntax"
[4]: ansible/playbooks_vault.html "Ansible Vault"

# [Inventory][1]

Ansible通过读取默认的节点配置文件/etc/ansible/hosts，可以同时连接到多个节点上执行任务。

当然，您也可以使用多个节点配置文件，以及[动态获取节点配置文件][2]。

## Hosts and Groups

/etc/ansible/hosts 是一种INI类型配置格式，如：

    mail.example.com

    [webservers]
    foo.example.com
    bar.example.com

    [dbservers]
    one.example.com
    two.example.com
    three.example.com

[]表示主机的分组名，可以按照功能、系统等进行分类，便于对某些主机或者某一组功能相同的主机进行操作。

一台主机属多个组是没有问题的，例如一台服务器既是网络服务器也是数据库服务器，所以可以放在webserver组和dbserver组。但需要注意的，主机上的属性变量会继承所有组的`vars`。

如果某些主机使用非标准的SSH端口，你可以在的主机名之后添加冒号及端口号。当SSH的配置文件中指定端口将不会使用paramiko进行连接，但可以使用openssh进行ssh连接。

为了简单明了，因此建议，不是使用默认22端口的机器，您可以如下设置：

    badwolf.example.com:5309

假如你想要为某些静态IP设置一些别名，或者通过tunnels进行连接。你可以这样定义：

    jumper ansible_port=5555 ansible_host=192.168.1.50

在上面的例子中，ansible可以通过别名“jumper”（不一定是真正的主机名）连接192.168.1.50端口5555。请注意，这是使用节点配置文件来定义一些特殊的变量。一般来说，这不是定义变量的最佳方式，后面还有其它方式。

需加入大量的主机？如果主机名类似以下模式，你可以仿照下面样例的做法，而不需要列出每个主机：

    [webservers]
    www[01:50].example.com

* 表示从www01到www50，共计50台主机

对于数字模式，可以根据需求使用前缀0，上面例子是包括0，表示www01、www02。若不使用0，可定义成www[1:50]，表示www1、www2。
您还可以使用字母定义可变范围，例如：

    [databases]
    db-[a:f].example.com

* 表示从db-a到db-f，共计7台主机

---

注意：

Ansible2.0已经把ansible_ssh_user，ansible_ssh_host和ansible_ssh_port精简成ansible_user，ansible_host和ansible_port。如果您使用Ansible2.0之前版本，你应该继续使用老式变量（ansible_ssh_*）。在Ansible旧版本中，不带“ssh”的变量会被忽略，没有任何警告。

---

对于每台主机的连接类型、连接用户等信息你都可以自定义，例如：

    [targets]

    localhost              ansible_connection=local
    other1.example.com     ansible_connection=ssh        ansible_user=mpdehaan
    other2.example.com     ansible_connection=ssh        ansible_user=mdehaan

对于上面的定义方式，仅仅是针对每台主机的一个快速定义，随后将会讨论如何在host_vars目录下的单个文件中定义。

## Host Variables

如上文提到，playbooks使用的变量，我们很容易就可以在单机或多台主机上定义：

    [atlanta]
    host1 http_port=80 maxRequestsPerChild=808
    host2 http_port=303 maxRequestsPerChild=909

## Group Variables

变量也可以通过组名应用到组内的所有成员：

    [atlanta]
    host1
    host2

    [atlanta:vars]
    ntp_server=ntp.atlanta.example.com
    proxy=proxy.atlanta.example.com


## Groups of Groups, and Group Variables

另外，定义主机组之间的继承关系使用`:children`作为后缀。就像上文，主机组内共同的变量使用`:vars`作为后缀：

    [atlanta]
    host1
    host2

    [raleigh]
    host2
    host3

    #southeast组包含atlanta和raleigh的机器
    [southeast:children]
    atlanta
    raleigh

    [southeast:vars]
    some_server=foo.southeast.example.com
    halon_system_timeout=30
    self_destruct_countdown=60
    escape_pods=2

    [usa:children]
    southeast
    northeast
    southwest
    northwest

如果你需要存储列表或hash数据，或需要从节点配置文件中分离出主机和组的具体变量，请参阅下一节。

## Splitting Out Host and Group Specific Data

Ansible并不建议变量存储在节点配置文件中。除了直接在主INI文件存储变量，主机或组的变量都可以存储在另外单独的文件中。
单独的变量文件必须遵循YAML语法。有效的文件扩展名包括“.yml”，“.yaml”，“.json”，或没有文件扩展名。关于YAML语法请参阅：[YAMLSyntax][3]

假设节点配置文件路径:

    /etc/ansible/hosts

如果主机名是“foosball”，两个组名分别是“raleigh”、“webservers”，以下路径的yaml文件中的变量将会应用到对应的主机上：

    /etc/ansible/group_vars/raleigh # 可使用扩展名是'.yml','.yaml','.json'
    /etc/ansible/group_vars/webservers
    /etc/ansible/host_vars/foosball

例如，您是通过服务器功能划分不同的机器到不同分组。raleigh组的机器有共同的变量，比如“/etc/ansible/group_vars/raleigh”内容：

    ---
    ntp_server: acme.example.org
    database_server: storage.example.org

当然即使上面文件不存在也是没有问题的，因为这是一项可选功能。

还有一种高级用法，您可以创建以组或主机的名字命名的目录，Ansible会读这些目录中的所有文件。请看例子：

    /etc/ansible/group_vars/raleigh/db_settings
    /etc/ansible/group_vars/raleigh/cluster_settings

所有在“raleigh”组的主机都有这些文件中定义的变量。当变量文件过大或当你想使用[Ansible Vault ][4]保存变量，使用这种方式组织变量都是非常有用。请注意，这仅适用于Ansible1.4或更高版本。

提示：在Ansible1.2或更高版本的group_vars/和host_vars/目录既可以在playbook所在目录或inventory目录中。如果两个路径存在，playbook目录变量将覆盖inventory目录中设置的变量。

提示：跟踪主机及变量变更的方案是把节点配置文件和变量存放在git（或其他版本控制系统）。

## List of Behavioral Inventory Parameters

正如上面提到的，ansible可以设置下面的变量控制与远程主机的连接参数。

Host connection:

    ansible_connection
      #与主机的连接方式。可选local, smart, ssh 或 paramiko。默认值是 smart.

---

注意：

Ansible2.0已经把ansible_ssh_user，ansible_ssh_host和ansible_ssh_port精简成ansible_user，ansible_host和ansible_port。如果您使用Ansible2.0之前版本，你应该继续使用老式变量（ansible_ssh_*）。在Ansible旧版本中，不带“ssh”的变量会被忽略，没有任何警告。

---

SSH 连接参数：

    ansible_host
      #连接主机的名称。你可以定义你想要的不同别名。
    ansible_port
      #ssh 端口号。如果不是22，需要定义。
    ansible_user
      #默认使用ssh 用户名。
    ansible_ssh_pass
      The ssh password to use (this is insecure, we strongly recommend using --ask-pass or SSH keys)
    ansible_ssh_private_key_file
      Private key file used by ssh.  Useful if using multiple keys and you don't want to use SSH agent.
    ansible_ssh_common_args
      This setting is always appended to the default command line for
      sftp, scp, and ssh. Useful to configure a ``ProxyCommand`` for a
      certain host (or group).
    ansible_sftp_extra_args
      This setting is always appended to the default sftp command line.
    ansible_scp_extra_args
      This setting is always appended to the default scp command line.
    ansible_ssh_extra_args
      This setting is always appended to the default ssh command line.
    ansible_ssh_pipelining
      Determines whether or not to use SSH pipelining. This can override the
      ``pipelining`` setting in ``ansible.cfg``.

升级特权(see :doc:`Ansible Privilege Escalation<become>` for further details)::

    ansible_become
      #相当于ansible_sudo或ansible_su，允许强制特权升级。
    ansible_become_method
      #允许设置特权升级方法。
    ansible_become_user
      Equivalent to ansible_sudo_user or ansible_su_user, allows to set the user you become through privilege escalation
    ansible_become_pass
      Equivalent to ansible_sudo_pass or ansible_su_pass, allows you to set the privilege escalation password

远程主机环境参数：

    ansible_shell_type
      The shell type of the target system. Commands are formatted using 'sh'-style syntax by default. Setting this to 'csh' or 'fish' will cause commands executed on target systems to follow those shell's syntax instead.
    ansible_python_interpreter
      The target host python path. This is useful for systems with more
      than one Python or not located at "/usr/bin/python" such as \*BSD, or where /usr/bin/python
      is not a 2.X series Python.  We do not use the "/usr/bin/env" mechanism as that requires the remote user's
      path to be set right and also assumes the "python" executable is named python, where the executable might
      be named something like "python26".
    ansible\_\*\_interpreter
      Works for anything such as ruby or perl and works just like ansible_python_interpreter.
      This replaces shebang of modules which will run on that host.

例子：

    some_host         ansible_port=2222     ansible_user=manager
    aws_host          ansible_ssh_private_key_file=/home/example/.ssh/aws.pem
    freebsd_host      ansible_python_interpreter=/usr/local/bin/python
    ruby_module_host  ansible_ruby_interpreter=/usr/bin/ruby.1.9.3


## seealso:

   :doc:`intro_dynamic_inventory`
       Pulling inventory from dynamic sources, such as cloud providers
   :doc:`intro_adhoc`
       Examples of basic commands
   :doc:`playbooks`
       Learning Ansible’s configuration, deployment, and orchestration language.
   `Mailing List <http://groups.google.com/group/ansible-project>`_
       Questions? Help? Ideas?  Stop by the list on Google Groups
   `irc.freenode.net <http://irc.freenode.net>`_
       #ansible IRC chat channel


