本文采用知识共享 署名-相同方式共享 4.0 国际 许可协议进行许可。
访问 <https://creativecommons.org/licenses/by-sa/4.0/> 查看该许可协议。

``` conf
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

# user nginx; # 进程运行用户
user nginx nginx; # 进程运行用户和用户组
worker_processes auto; # 进程数 auto/x 建议 CPU 线程数 + 1
worker_rlimit_nofile 65535; # 单进程最多打开的文件描述符数，由于请求分配不会均匀，建议直接设成与 ulimit -n 一致
#error_log /var/log/nginx/error.log; # 全局错误日志
error_log /var/log/nginx/error.log info; # 全局错误日志 日志类型[ debug | info | notice | warn | error | crit ]
pid /run/nginx.pid; # 进程文件

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf; # 导入其他位置配置文件

events {
    # 参考事件模型, 各类型 OS 支持的模型不同, Linux 推荐 epoll
    # [ kqueue | rtsig | epoll | /dev/poll | select | poll ]
    use epoll; 
    # 单进程最大连接数，Linux 受 Kernel sys.fs.file-max 限制
    # 修改: ulimit -SHn 65535
    worker_connections 10240;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;

    # Gzip
    gzip                on; # 开启 Gzip 压缩输出
    gzip_min_length     1k; # 设置允许压缩的页面最小字节数，页面字节数从header头得content-length中进行获取。默认值是0，不管页面多大都压缩。建议设置成大于2k的字节数，小于2k可能会越压越大。
    gzip_buffers 4      16k;# 设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。 例如 4 4k 代表以4k为单位，按照原始数据大小以4k为单位的4倍申请内存。 4 8k 代表以8k为单位，按照原始数据大小以8k为单位的4倍申请内存。 如果没有设置，默认值是申请跟原始数据相同大小的内存空间去存储gzip压缩结果。
    gzip_http_version 1.0; # 压缩版本（默认1.1，前端如果是squid2.5请使用1.0）
    gzip_comp_level 2; # 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间
    #gzip_types text/html
    gzip_types text/plain application/x-javascript text/css application/xml; # 压缩类型，默认就已经包含text/html，所以下面就不用再写了，写上去也不会有问题，但是会有一个warn。
    gzip_disable "MSIE [1-6]\."; # E6及以下禁止压缩
    gzip_vary on; # 给CDN和代理服务器使用，针对相同url，可以根据头信息返回压缩和非压缩副本

    # Sendfile
    sendfile            on; # 是否调用 sendfile 函数传输文件，当磁盘 IO 压力大时可以适当关闭
    tcp_nopush          on; # 在 Linux/Unix 中优化 TCP 传输，依赖 sendfile 选项
    tcp_nodelay         on; # 将小包组成大包, 提高带宽利用率(Nagle 算法)
    autoindex off; # 开启列表访问，通常结合 fancyindex 美化目录样式
    keepalive_timeout   65; # 长连接超时时间
    types_hash_max_size 2048; # 影响散列表的冲突率，与内存占用成正比，当报错 "无法构建types_hash" 时酌情修改

    # 浏览器只识别 mime_type 来展示内容，Nginx 可以处理响应文件拓展名与 mime_type 之间的关联
    include             /etc/nginx/mime.types; # 引入 mime_type 关联表
    default_type        application/octet-stream; # 默认 mime_type 类型
    #charset utf-8; # 默认编码
    client_max_body_size 20M; # 请求体 Max 大小，通常用于上传文件大小限制

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf; # 引入外部配置文件，通常用于网站之间解耦合，即一个网站一个配置文件

    server {
        listen       80 default_server; # 监听端口，default 即请求 80 端口时默认匹配此 Server
        listen       [::]:80 default_server; # IPV6 监听
        server_name  _; # 监听域名/主机名
        root         /usr/share/nginx/html; # Server 主目录，在 location 没有匹配时会继续匹配此 Path 下的文件
        index index.html index.htm index.jsp; # 默认页面匹配
        #charset utf-8; # 编码

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf; # 引入一些默认配置

        upstream httpds { # 反向代理 Stream
            server 127.0.0.1:8080    weight=5  max_conns=800 max_fails=1  fail_timeout=20;
        }

        # location [ = | ~ | * | ^ ] uri { ... }
        # location URI {} 对当前路径及子路径下的所有对象都生效；
        # location = URI {} 注意URL最好为具体路径。 精确匹配指定的路径，不包括子路径，因此，只对当前资源生效；
        # location ~ URI {} location ~* URI {} 模式匹配URI，此处的URI可使用正则表达式，区分字符大小写，*不区分字符大小写；
        # location ^~ URI {} 禁用正则表达式
        # 优先级：= > ^~ > |* > /|/dir/
        location / {
            # IP 访问控制
            #deny  IP / IP段
            deny  172.16.9.5;
            allow 172.16.9.0/24;192.168.0.0/16;192.0.0.0/8

            # 用户 Auth
            # 生成用户配置文件并将 wars 用户追加: htpasswd -c -d /etc/nginx/conf.d/users wars
            auth_basic  "closed site"; # 登陆提示
            auth_basic_user_file /etc/nginx/conf.d/users; # 用户配置文件

            # Nginx 访问状态监控页面
            stub_status on;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }

        access_log  "pipe:rollback logs/host.access_log interval=1d baknum=7 maxsize=2G"  main;
    }
}
```
