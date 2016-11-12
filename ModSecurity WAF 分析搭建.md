#ModSecurity WAF 分析搭建
关于waf,很多安全人士和黑客都不陌生，最近菜鸟黑客跳跳龙正在研究waf的相关知识，想从防守一方去更好的理解安全,这里我选择了`ModSecurity`安全模块，跟`nginx`搭配，加上`docker centos`操作系统，完成了简易的waf系统的搭建，并且实现了一些常见的非法不安全的http访问的拦截。


##WAF搭建过程
这里我使用centos的原因是之前基于debian的一些依赖和模块安装起来经常失败,找不到对应的安装包。恰好网络上很多都是基于centos的实例，相关的二进制安装包也很多，最后利用docker的端口映射完成了测试

##需要的资源（可直接wget）

* modsecurity for Nginx ：https://www.modsecurity.org/tarball/2.8.0/modsecurity-2.8.0.tar.gz
* OWASP规则 ： https://github.com/SpiderLabs/owasp-modsecurity-crs

首先把这些资源wget到本地，除了上面的这两个，还需要nginx服务器，可以去官网直接下载源码到同目录，先不要编译。

```
[root@7758028ea7f2 temp]# ls -al
total 4660
drwxr-xr-x  5 root root    4096 Nov 10 11:28 .
drwxr-xr-x 17 root root    4096 Nov 10 13:29 ..
drwx------ 14  119  128    4096 Nov 10 11:12 modsecurity-2.8.0
-rw-r--r--  1 root root 3940357 Jun 19  2014 modsecurity-2.8.0.tar.gz
drwxr-xr-x  9 1001 1001    4096 Nov 10 11:14 nginx-1.6.1
-rw-r--r--  1 root root  803301 Aug  5  2014 nginx-1.6.1.tar.gz
drwxr-xr-x 10 root root    4096 Nov 10 11:29 owasp-modsecurity-crs
```
安装之前先解决环境依赖，ModSecurity for Nginx 需要的依赖

* pcre 
* httpd-devel 
* libxml2 
* apr

```
yum install httpd-devel apr apr-util-devel apr-devel  pcre pcre-devel  libxml2 libxml2-devel
```
启用standalone模块并编译,进入ModSecurity

```
./autogen.sh
./configure --enable-standalone-module --disable-mlogc
make 
```

编译nginx，并且添加 modsecurity

```
./configure --add-module=../modsecurity-2.8.0/nginx/modsecurity/  
make && make install
```

最好就是规则的处理了，这也是ModSecurity的强大之处
首先重命名modsecurity_crs_10_setup.conf.example文件

```
cd owasp-modsecurity-crs&& mv modsecurity_crs_10_setup.conf.example modsecurity_crs_10_setup.conf
```
启用OWASP规则：
复制modsecurity源码目录下的modsecurity.conf-recommended和unicode.mapping到nginx的conf目录下，并将modsecurity.conf-recommended重新命名为modsecurity.conf。
其中nginx的安装目录在usr/local/nginx
编辑modsecurity.conf 文件，将SecRuleEngine设置为 on
owasp-modsecurity-crs下有很多存放规则的文件夹，例如base_rules、experimental_rules、optional_rules、slr_rules，里面的规则按需要启用，需要启用的规则使用Include进来即可。

```
Include owasp-modsecurity-crs/modsecurity_crs_10_setup.conf
Include owasp-modsecurity-crs/base_rules/modsecurity_crs_41_sql_injection_attacks.conf
Include owasp-modsecurity-crs/base_rules/modsecurity_crs_41_xss_attacks.conf
Include owasp-modsecurity-crs/base_rules/modsecurity_crs_40_generic_attacks.conf
Include owasp-modsecurity-crs/experimental_rules/modsecurity_crs_11_dos_protection.conf
Include owasp-modsecurity-crs/experimental_rules/modsecurity_crs_11_brute_force.conf
Include owasp-modsecurity-crs/optional_rules/modsecurity_crs_16_session_hijacking.conf
```

在需要启用modsecurity的主机的location下面加入
ModSecurityEnabled on; 
ModSecurityConfig modsecurity.conf;

到这里安装就结束了，最后看一下一些配置文件(nginx/conf)

```
[root@7758028ea7f2 conf]# ls -al
total 140
drwxr-xr-x  3 root root  4096 Nov 10 12:46 .
drwxr-xr-x 13 root root  4096 Nov 10 13:29 ..
-rw-r--r--  1 root root  1034 Nov 10 11:24 fastcgi.conf
-rw-r--r--  1 root root  1034 Nov 10 11:24 fastcgi.conf.default
-rw-r--r--  1 root root   964 Nov 10 11:24 fastcgi_params
-rw-r--r--  1 root root   964 Nov 10 11:24 fastcgi_params.default
-rw-r--r--  1 root root  2837 Nov 10 11:24 koi-utf
-rw-r--r--  1 root root  2223 Nov 10 11:24 koi-win
-rw-r--r--  1 root root  3957 Nov 10 11:24 mime.types
-rw-r--r--  1 root root  3957 Nov 10 11:24 mime.types.default
-rw-------  1 root root  8969 Nov 10 12:42 modsecurity.conf
-rw-r--r--  1 root root  2719 Nov 10 12:46 nginx.conf
-rw-r--r--  1 root root  2656 Nov 10 11:24 nginx.conf.default
drwxr-xr-x 10 root root  4096 Nov 10 12:33 owasp-modsecurity-crs
-rw-r--r--  1 root root   596 Nov 10 11:24 scgi_params
-rw-r--r--  1 root root   596 Nov 10 11:24 scgi_params.default
-rw-------  1 root root 53642 Nov 10 12:38 unicode.mapping
-rw-r--r--  1 root root   623 Nov 10 11:24 uwsgi_params
-rw-r--r--  1 root root   623 Nov 10 11:24 uwsgi_params.default
-rw-r--r--  1 root root  3610 Nov 10 11:24 win-utf
```

```
[root@7758028ea7f2 conf]# cat modsecurity.conf 
# -- Rule engine initialization ----------------------------------------------

# Enable ModSecurity, attaching it to every transaction. Use detection
# only to start with, because that minimises the chances of post-installation
# disruption.
#
SecRuleEngine On


# -- Request body handling ---------------------------------------------------

........
........
........

Include owasp-modsecurity-crs/modsecurity_crs_10_setup.conf
Include owasp-modsecurity-crs/base_rules/modsecurity_crs_41_sql_injection_attacks.conf
Include owasp-modsecurity-crs/base_rules/modsecurity_crs_41_xss_attacks.conf
Include owasp-modsecurity-crs/base_rules/modsecurity_crs_40_generic_attacks.conf
Include owasp-modsecurity-crs/experimental_rules/modsecurity_crs_11_dos_protection.conf
Include owasp-modsecurity-crs/experimental_rules/modsecurity_crs_11_brute_force.conf
Include owasp-modsecurity-crs/optional_rules/modsecurity_crs_16_session_hijacking.conf


```

```
[root@7758028ea7f2 conf]# cat nginx.conf
server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

	ModSecurityEnabled on;  
	ModSecurityConfig modsecurity.conf;
        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
.........
.........
.........
```



