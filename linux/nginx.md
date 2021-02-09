nginx in ubuntu20.04

> https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms
> https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04


### 安装命令
`$ sudo apt update`
`$ sudo apt install nginx`

### 查看防火墙
`$ sudo ufw app list`
`$ sudo ufw status`

### 查看nginx
`$ systemctl status nginx`
`$ curl -4 icanhazip.com`
`$ curl http://your_server_ip`

### 管理nginx
`sudo systemctl stop nginx`
`sudo systemctl start nginx`
`sudo systemctl restart nginx`
`sudo systemctl reload nginx`
`sudo systemctl disable nginx`
`sudo systemctl enable nginx`

### 设置服务块
ubuntu20.04上的Nginx有一个默认启用的服务器块，它被配置为在/var/www/html目录下提供文档。虽然这对单个站点很有效，但如果您托管多个站点，它可能会变得很不方便。与其修改/var/www/html，不如在/var/www中为您的域站点创建一个目录结构，如果客户端请求与任何其他站点不匹配，则保留/var/www/html作为默认目录。

`$ sudo mkdir -p /var/www/your_domain/html`
`$ sudo chown -R $USER:$USER /var/www/your_domain/html`
`$ sudo chmod -R 755 /var/www/your_domain`
`$ echo "OK!" > /var/www/your_domain/html/index.html`

为了让Nginx提供这些内容，有必要使用正确的指令创建一个服务器块。与其直接修改默认配置文件，不如在/etc/nginx/sites-available/your_domain中创建一个新的配置文件：
```nginx
server {
        listen 80;
        listen [::]:80;

        root /var/www/your_domain/html;
        index index.html index.htm index.nginx-debian.html;

        server_name your_domain www.your_domain;

        location / {
                try_files $uri $uri/ =404;
        }
}
```
接下来，让我们通过创建一个链接来启用该文件，该链接指向启用了站点的目录，Nginx在启动时从中读取该目录：
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/

为了避免由于添加两个server_name而可能出现的哈希桶内存问题，有必要调整/etc/nginx/nginx.conf文件中的配置：
```nginx
...
http {
    ...
    server_names_hash_bucket_size 64;
    ...
}
...
```
`$ sudo nginx -t`
`$ sudo systemctl restart nginx`


既然您已经知道如何管理Nginx服务本身，那么您应该花几分钟时间熟悉一些重要的目录和文件。


/var/www/html: 实际的web内容，默认情况下仅由您之前看到的默认Nginx页面组成，由/var/www/html目录提供。这可以通过改变Nginx配置文件来改变。
/etc/nginx: nginx配置目录。所有Nginx配置文件都驻留在这里。
/etc/nginx/nginx.conf: Nginx主配置文件。可以对其进行修改以更改Nginx全局配置。
/etc/nginx/sites-available: 可以存储每个站点服务器块的目录。Nginx不会使用此目录中的配置文件，除非它们链接到启用站点的目录。通常，所有服务器块配置都在这个目录中完成，然后通过链接到另一个目录来启用。
/etc/nginx/sites-enabled: 存储每个站点启用的服务器块的目录。通常，这些是通过链接到站点可用目录中的配置文件来创建的。
/etc/nginx/snippets: 这个目录包含可以包含在nginx配置中其他地方的配置片段。潜在的可重复配置段很适合重构为代码段。
/var/log/nginx/access.log: 对web服务器的每个请求都记录在此日志文件中，除非Nginx配置为执行其他操作。
/var/log/nginx/error.log: 任何Nginx错误都将记录在此日志中。

## 首先，Nginx查看请求的IP地址和端口。它将此与每个服务器的listen指令相匹配，以构建可能解析请求的服务器块列表。

listen指令通常定义服务器块将响应的IP地址和端口。默认情况下，任何不包含listen指令的服务器块都被赋予listen参数0.0.0.0:80（如果Nginx是由正常的非root用户运行的，则为0.0.0.0:8080）。这允许这些块响应端口80上任何接口上的请求，但此默认值在服务器选择过程中没有太大的作用。
listen指令可以设置为：

- IP地址/端口组合。
- 一个单独的IP地址，然后在默认端口80上侦听。
- 一个单独的端口，它将侦听该端口上的每个接口。
- Unix套接字的路径。

**最后一个选项通常只在不同服务器之间传递请求时才有意义。**

当尝试确定向哪个服务器块发送请求时，Nginx将首先尝试使用以下规则根据listen指令的特定性来决定：

1. Nginx通过用缺省值替换丢失的值来翻译所有“不完整”的listen指令，这样每个块都可以通过其IP地址和端口进行计算。下面是一些实例：
- 没有listen指令的块使用默认值0.0.0.0:80。
- 设置为IP地址111.111.111.111（没有端口）的块变为111.111.111.111:80
- 设置为没有IP地址的端口8888的块变为0.0.0.0:8888

2. 然后Nginx尝试收集一个服务器块列表，这些服务器块与基于IP地址和端口的请求最为匹配。这意味着，如果存在列出特定IP地址的匹配块，则在功能上使用0.0.0.0作为其IP地址（以匹配任何接口）的任何块都不会被选中。无论如何，端口必须完全匹配。

3. 如果只有一个最特定的匹配项，则该服务器块将用于服务请求。如果有多个服务器块具有相同级别的特定性匹配，Nginx将开始计算每个服务器块的server_name指令。

重要的是要理解，Nginx只会在需要区分与listen指令中的同一特定级别匹配的服务器块时才计算server_name指令。例如，如果example.com网站托管在192.168.1.10的端口80上，请求example.com网站将始终由本例中的第一个块提供服务，尽管第二个块中有server_name指令。

```nginx
server {
    listen 192.168.1.10;
}

server {
    listen 80;
    server_name example.com;
}
```

## 接下来，为了进一步评估具有同样特定的listen指令的请求，Nginx检查请求的“Host”头。此值保存客户端实际尝试访问的域或IP地址。

- Nginx将首先尝试找到一个服务器块，其服务器名称与请求的“Host”头中的值完全匹配。如果找到这个，关联的块将用于服务请求。如果找到多个完全匹配，则使用第一个。
- 如果找不到完全匹配的服务器，Nginx将尝试使用前导通配符（在配置中名称的开头用\*表示）查找具有匹配的server_name的服务器块。如果找到一个，则该块将用于服务请求。如果找到多个匹配项，则最长的匹配项将用于服务请求。
- 如果使用前导通配符找不到匹配项，Nginx将查找具有server_name且使用尾随通配符匹配的服务器块（由配置中以\*结尾的服务器名称表示）。如果找到一个，则该块用于服务请求。如果找到多个匹配项，则最长的匹配项将用于服务请求。
- 如果使用尾随通配符找不到匹配项，Nginx将使用正则表达式（在名称前用\~表示）计算定义server_name的服务器块。第一个server_name（带有与“Host”头匹配的正则表达式）将用于服务请求。

### 实例：
修改/etc/hosts

```bash
192.168.0.198 example.com
192.168.0.198 www.example.com
192.168.0.198 abc.example.com
192.168.0.198 www.example.cn
192.168.0.198 www.example.org
192.168.0.198 www.abc.org
```


192.168.0.198 nginx.conf
```nginx
# example.com:8080 找不到匹配的server_name 使用第一个匹配端口的server块
server {
    listen 8080;
    server_name localhost;
    location / {
        echo "localhost";
    }
}

# www.example.com:8080 完全匹配
server {
    listen 8080;
    server_name www.example.com;
    location / {
        echo "www.example.com";
    }
}

# abc.example.com:8080
server {
    listen 8080;
    server_name *.example.com;
    location / {
        echo "*.example.com";
    }
}

# www.example.cn:8080
server {
    listen 8080;
    server_name www.example.*;
    location / {
        echo "www.example.*";
    }
}

# www.abc.org:8080
server {
    listen 8080;
    server_name *.org;
    location / {
        echo "*.org"; 
    }
}

# www.example.org:8080
server {
    listen 8080;
    server_name *.example.org;
    location / {
        echo "*.example.org";
    }
}

server {
    listen 9800;
    server_name example.com;
    location / {
        echo "first";
    }
}

# www.example.com:9800 使用第一个匹配的正则表达式响应请求
server {
    listen 9800;
    server_name ~^(www|host1).*\.example\.com$;
    location / {
        echo "second";
    }
}


server {
    listen 9800;
    server_name ~^(subdomain|set|www|host1).*\.example\.com$;
    location / {
        echo "third";
    }
}
```
