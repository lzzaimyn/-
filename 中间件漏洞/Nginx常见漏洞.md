Nginx常见漏洞
nginx的HTTP和反向代理web服务器，同时也提供了IMAP/POP3/SMTP服务
目录结构
[root@localhost ~]# tree /usr/local/nginx
/usr/local/nginx
├── client_body_temp                 # POST 大文件暂存目录
├── conf                             # Nginx所有配置文件的目录
│   ├── fastcgi.conf                 # fastcgi相关参数的配置文件
│   ├── fastcgi.conf.default         # fastcgi.conf的原始备份文件
│   ├── fastcgi_params               # fastcgi的参数文件
│   ├── fastcgi_params.default       
│   ├── koi-utf
│   ├── koi-win
│   ├── mime.types                   # 媒体类型
│   ├── mime.types.default
│   ├── nginx.conf                   #这是Nginx默认的主配置文件，日常使用和修改的文件
│   ├── nginx.conf.default
│   ├── scgi_params                  # scgi相关参数文件
│   ├── scgi_params.default  
│   ├── uwsgi_params                 # uwsgi相关参数文件
│   ├── uwsgi_params.default
│   └── win-utf
├── fastcgi_temp                     # fastcgi临时数据目录
├── html                             # Nginx默认站点目录
│   ├── 50x.html                     # 错误页面优雅替代显示文件，例如出现502错误时会调用此页面
│   └── index.html                   # 默认的首页文件
├── logs                             # Nginx日志目录
│   ├── access.log                   # 访问日志文件
│   ├── error.log                    # 错误日志文件
│   └── nginx.pid                    # pid文件，Nginx进程启动后，会把所有进程的ID号写到此文件
├── proxy_temp                       # 临时目录
├── sbin                             # Nginx 可执行文件目录
│   └── nginx                        # Nginx 二进制可执行程序
├── scgi_temp                        # 临时目录
└── uwsgi_temp                       # 临时目录

0x01 文件解析漏洞
原理：该漏洞与nginx、php版本无关，属于用户配置不当造成的解析漏洞，于任意文件名，在后面添加/xxx.php（xxx）为任意字符后，即可将文件作为php解析。
修复：将php.ini文件中的cgi.fix_pathinfo的值设置为0   php-fpm.conf中的security.limit_extensions后面的值设置为.php

0x02 目录遍历漏洞
原理：Nginx的目录遍历与apache一样,属于配置方面的问题,错误的配置可导致目录遍历与源码泄露。 autoindex 被配置为on了
修复：将autoindex改为off

0x03 空字节任意代码执行漏洞
原理：Ngnix在遇到%00空字节时与后端FastCGI处理不一致，导致可以在图片中嵌入PHP代码然后通过访问xxx.jpg%00.php来执行其中的代码.
例如请求  GET /test.php%00.jpg HTTP/1.1
          Host: example.com**
 这个请求看起来像是请求一张名为“test.php.jpg”的图片，但是由于包含了空字节，Nginx会将其解析为请求“test.php”，这样攻击者就可以执行任意PHP代码。
影响版本
nginx 0.5.xx
nginx 0.6.xx
nginx 0.7 <= 0.7.65
nginx 0.8 <= 0.8.37
首先上传写入木马的1.jpg ，然后访问1.jpg..php，再抓包放到重放器模块将hex选项卡中的jpg后面的一个"." 修改为00就成功绕过。


0x03 CRLF注入漏洞
原理：
CRLF是”回车+换行”(rn)的简称,其十六进制编码分别为0x0d和0x0a。在HTTP协议中，HTTP header与HTTP Body是用两个CRLF分隔的，浏览器就是根据这两个CRLF来取出HTTP内容并显示出来。
所以,一旦我们能够控制HTTP消息头中的字符，注入一些恶意的换行,这样我们就能注入一些会话Cookie或者HTML代码。CRLF漏洞常出现在Location与Set-cookie消息头中。
访问链接：http://xx.xx.xx.xx/%0ASet-cookie:JSPSESSID%3D123
（换行和回车的URL编码分别是%0d%0a）
抓包，可以看到将6666通过set-cookie返回；可以配合xss利用

0x04 文件名逻辑漏洞(CVE-2013-4547)
原理：当请求如下URI时：/test[0x20]/…/admin/index.php，这个URI不会匹配上location后面的/admin/，也就绕过了其中的IP验证；但最后请求的是/test[0x20]/…/admin/index.php文件，也就是/admin/index.php，成功访问到后台。（这个前提是需要有一个目录叫test：这是Linux系统的特点，如果有一个不存在的目录，则即使跳转到上一层，也会爆文件不存在的错误，Windows下没有这个限制）
举个例子，比如，Nginx 匹配到.php 结尾的请求，就发送给 fastcgi 进行解析，常见的写法如下：
location ~ \.php$ {
    include        fastcgi_params;
    fastcgi_pass   127.0.0.1:9000;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  /var/www/html$fastcgi_script_name;
    fastcgi_param  DOCUMENT_ROOT /var/www/html;
}
正常情况下（关闭 pathinfo 的情况下），只有.php 后缀的文件才会被发送给 fastcgi 解析。
而存在 CVE-2013-4547 的情况下，我们请求 1.gif[0x20][0x00].php，这个 URI 可以匹配上正则 \.php$，可以进入这个 Location 块；但进入后，Nginx 却错误地认为请求的文件是 1.gif[0x20]，就设置其为 SCRIPT_FILENAME 的值发送给 fastcgi。
fastcgi 根据 SCRIPT_FILENAME 的值进行解析，最后造成了解析漏洞。

首先上传1.jpg，并进行抓包，在数据包1.jpg后添加一个空格，再访问返回的该链接并加.php抓包；http://x.x.x.x:8080/uploadfiles/1.jpg...php 在hex选项卡中将jpg后面的2个点的hex值2e分别修改为20，00；成功绕过。
版本：Nginx 0.8.41 ~ 1.4.3 / 1.5.0 ~ 1.5.7




