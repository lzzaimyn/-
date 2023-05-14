Apache漏洞
apache的目录结构：
bin 	存在常用命令工具，例如：start.bat、httpd.bat
cgi-bin	存放linux下常用的命令。例如：xxx.sh
conf	apache的相关配置文件，例如：httpd.conf
error	错误日志记录
htdocs	放网站源码的地方
logs	日志
manual	手册
modules	扩展模块

0x01 未知扩展名解析漏洞
原理：apache默认一个文件可以有多个以点分割的后缀，比如test.php.xxx，当最右边的后缀（xxx）无法识别（不在mime.types文件内），则继续向左识别，知道识别到合法后缀才能进行解析

0x02 apache httpd换行解析漏洞（CVE-2017-15715)
原理：正则表达式在结尾处$ 符号，如果设置了 RegExp 对象的 Multiline 属性，则 $ 也匹配 '\n' 或 '\r'。因为 1.php\x0a = 1.php\n，所以我们在上传文件名后面加上\0xa（换行符），也会以php文件形式解析执行，该漏洞属于用户配置不当产生的漏洞，与具体中间件版本无关。
版本：2.4.0~2.4.29版本

0x03 Apche SSI远程命令执行漏洞（ssi-rce）
SSI（服务器端包含）是放置在HTML页面中的指令，并在服务页面时在服务器上对其进行评估。使用 SSI 可以动态的创建一部分网页内容而不需要编写复杂的 JSP/ASP/PHP 等程序。
从技术角度上来说，SSI就是在HTML文件中，可以通过注释行调用的命令或指针，即允许通过在HTML页面注入脚本或远程执行任意命令。
SSI可以完成查看时间、文件修改时间、CGI程序执行结果、执行系统命令、连接数据库等操作，功能很强大。
利用：在测试任意文件上传漏洞的时候，目标服务端可能不允许上传php后缀的文件。如果目标服务器开启了SSI与CGI支持，我们可以上传一个shtml文件，并利用<!--#exec cmd="ls /" -->语法执行任意命令。默认扩展名是 .stm、.shtm 和 .shtml。

不允许上传php文件的情况下，创建脚本1.shtml
<pre>
<!--#exec cmd="ls" -->
</pre>
访问文件成功执行

0x04 apache shiro反序列化漏洞
Apache Shiro 是一个强大易用的Java安全框架,提供了认证、授权、加密和会话管理等功能，Shiro框架直观、易用、同时也能提供健壮的安全性。
Apache Shiro反序列化漏洞分为两种：Shiro-550、Shiro-721

Apache Shiro框架提供了记住密码的功能（RememberMe），用户登录成功后会生成经过加密并编码携带用户信息的cookie。在服务端对rememberMe的cookie值，先base64解码然后AES解密再反序列化，就导致了反序列化RCE漏洞。
那么，Payload产生的过程：
命令=>序列化=>AES加密=>base64编码=>RememberMe Cookie值
在整个漏洞利用过程中，比较重要的是AES加密的密钥，如果没有修改默认的密钥那么就很容易就知道密钥了,Payload构造起来也是十分的简单。
版本：Apache Shiro < 1.2.4
Shiro反序列化的流量特证：
返回包中会包含rememberMe=deleteMe字段
这种情况大多会发生在登录处，返回包里包含remeberMe=deleteMe字段，这个是在返回包中
如果返回的数据包中没有remeberMe=deleteMe字段的话，可以在数据包中的Cookie中添加remeberMe=deleteMe字段这样也会在返回包中有这个字段。




