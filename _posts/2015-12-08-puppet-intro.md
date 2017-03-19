---
layout: post
title: "[openstack] puppet整体介绍"
comments: true
categories:
- openstack
---

puppet安装
---------

- 每台机器都要设置好hostname，因为ssl证书需要。
- NTP保持时间一致
- client端安装puppet包
- master端安装puppetmaster包

puppet注意点
--------------

- 是C/S架构
- client会定期30分钟和master通信，如果配置信息被修改，则执行。间隔修改的方法： /etc/puppet/puppet.conf中的[agent]中设置 runinterval = 60
- 也可以人工强制执行: puppet agent -t

client与master的交互过程
----------------------

1. 客户端puppet调用facter，facter会探测出这台主机的一些变量如主机名、内存大小、IP地址等。然后puppet把这些信息发送到服务器端。
1. 服务器端的puppetmaster检测到客户端的主机名，然后会到manifest里面对应的node配置，然后对这段内容进行解析，
facter送过来的信息可以作为变量进行处理的，node牵涉到的代码才解析，其它的代码不解析，解析分几个过程：
语法检查、然后会生成一个中间的伪代码，然后再把伪代码发给客户机。
1. 客户端接收到伪代码之后就会执行，客户端再把执行结果发送给服务器。
1. 服务器再把客户端的执行结果写入日志。


语法
----

- 变量

  - 声明格式：$变量名="值"
  - 引用格式：${变量名}
  - 如果要取facts中的值，可以使用 ${::fact-name} ,这种写法告诉人们是top-scope variable，而不是用户自己定义的local-scope variable

- 字符串
  - '不包含变量的字符串用单引号'
  - "包含变量的字符串需要用双引号 ${::operatingsystem} ，同时变量要用花括号包裹起来"
  - 对变量本身的引用不需要单引号或双引号。 例如  ${::operatingsystem}

- 数组

  - [ "apache2" , " httpd " , " ssh " ]

- 资源：

  ```
  type { "title ":
    attribute =>"value",
    ...
    attribute => "value",
  }
  ```

  注意：

  - type不能乱写，puppet已定义了很多type，例如：
    - file
    - package
    - service
    - cron
    - user
    - group
    - exec
    - notify
  - attribute =>"value"写法
  - 属性间“，”分割

- 引用
  首字母大写是`引用`，小写是`声明`
- 资源之间的关系
  - before
  - after
  - require
- class：资源的集合

  ```
  class 类名 {

      type { "title ":
  　　　　attribute => "value",
         ...
         attribute => "value",
  　　 }

      type { "title ":
  　　　　attribute => "value",
         ...
         attribute => "value",
  　　 }
  }
  ```

- class继承

  ```
  class 类名（新建）inherits 父类名（已存在）{
         Type ["title"] {attribute => "value",}
  }
  ```

- 函数定义

  ```
  define 函数名(变量名1,...,变量名n) {         #格式：$var
         type { "title ":
  　　　　       attribute => "变量名",       #格式：${var},下同
                ...
                attribute => "变量名",
  　　       }

         type { "title ":
  　　　　       attribute => "变量名",
                ...
                attribute => "变量名",
  　　       }
  }
  ```

- 引用define

  ```
  函数名 {
         变量名 => "值",
         ...
         变量名 => "值",
  }
  ```

  ```
  类名::函数名 {
         变量名 => "值",
         ...
         变量名 => "值",
  }
  ```

- 模块

  ```
  up-testmodule   --模块名
  ├── files       --pp中需要用到的文件
  ├── manifests   --所有pp文件
  │   └── init.pp --入口，在其中引用其他pp
  ├── Modulefile
  ├── templates    --xxx.erb    
  ```

- 模块中的namespace

  例如mysql::server

- 模板template
    - puppet通过erb(embedded Ruby）支持模板，模板的文件名必须以erb结尾
    - 检查模板： `erb -x -T '-' xxx.erb | ruby -c`
    - 可以使用数组、条件语句、变量作用域
- 好的命名规范
  - 模块名就是软件名字，例如apache
  - 类名应该是模块名::功能名，例如 `apache::vhost`
- node
  - 建议专门新建nodes.pp来存放node信息
  - 默认节点定义

    ```
    node default {
        变量的声明                  #声明格式：$变量名="值" 引用格式： ${变量名}
        include 类名,...,类名       #已定义好的类class
    }
    ```  


自己编写facter的方法
------------------

举例：/modules/haproxy/lib/facter/haproxy_version.rb

当agent apply haproxy时，就在本地增加了自定义的facter，该facter可以收集agent上的haproxy的版本号

facter -p haproxy_version 可以查看该facter值


最佳实践
-------

调试方法：

-
```
Exec {'output': logoutput => on_failure}
```

-
```
notify { "aaa ${fqdn} "}
```

生成例子：

```
puppet resource file /etc/keystone/policy.json
file { '/etc/keystone/policy.json':
  ensure  => 'file',
  content => '{md5}459ce538f825ed818c7794beeddf4523',
  ctime   => '2016-03-02 09:06:02 +0100',
  group   => '163',
  mode    => '640',
  mtime   => '2016-03-02 08:20:02 +0100',
  owner   => '163',
  type    => 'file',
}
```

例子： puppet client在本地执行manifest
------------------------------

- 文件管理

    1. 写一个manifest： test.pp

        ```
        file {
            "/tmp/haha":
            content => "haha"
        }
        ```

    1. 执行 `puppet apply test.pp`
    1. 在/tmp目录下会发现新文件

- 包管理

    1. 写一个manifest： test.pp

        ```
        package {
            ["gcc", "make", "nginx", "haproxy"]:
            ensure => installed
        }
        ```

    1. 执行 `sudo puppet apply test.pp`
    1. 上述4个包会被自动安装
