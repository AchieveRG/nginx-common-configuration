## Nginx common useful configuration

[![Try in Codesandbox.io](https://img.shields.io/badge/Try%20in%20Codesandbox.io-blue)](https://codesandbox.io/p/github/tldr-devops/nginx-common-configuration/master?file=/.codesandbox/README.md:1,1)

Nginx configs. Not the most powerful, productive or the best one. Just useful configs, which I would like to see in default nginx packages out of the box 😆
Bonus: fail2ban, filebeat, dockerfile and docker-compose configs for nginx :)

**Motivation**: I have been using nginx since 2015, and I configured it really for hundreds setups of 30+ companies and startups: sites, apps, websockets, proxies, load balancing, from a few up to 1k rps, etc... And I'm a little bit disappointed by [the official nginx wiki](https://www.nginx.com/resources/wiki/).
The last drop was this [blog post in the official blog](https://www.nginx.com/blog/help-the-world-by-healing-your-nginx-configuration/):
this post doesn't provide a complete solution, half of these tips can be included into nginx configs or snippets by default,
and some of the other tips, such as disabling access logging, in my opinion are the bad practice 😆

At the same time there are a lot good documentation and best practices:
[nginx docs](https://nginx.org/en/docs/),
[digitalocean config generator](https://www.digitalocean.com/community/tools/nginx),
[mozilla ssl best practices](https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=intermediate&openssl=1.1.1d&guideline=5.4),
etc...

And there are also some more interesting projects and examples:
- [nginx-admins-handbook](https://github.com/trimstray/nginx-admins-handbook)  
Huge total guide, must read for any nginx admin.
- [html5-boilerplate nginx configs](https://github.com/h5bp/server-configs-nginx)  
Most popular collection of configuration snippets.
- [nginx-boilerplate](https://github.com/nginx-boilerplate/nginx-boilerplate)  
Another one common boilerplate.
- [elasticweb/nginx-configs](https://github.com/elasticweb/nginx-configs)  
Collection of Nginx configs for most popular CMS/CMF/Frameworks based on PHP.
- [openbridge/nginx](https://github.com/openbridge/nginx)  
Docker image, but I haven't checked it properly yet, their configs require additional nginx modules and setup
and it can't be just copied to the usual nginx setup. However, you can use it with docker.
Also I don't agree with nginx microcache for every site, see known traps.
- [hub.docker.com/_/nginx](https://hub.docker.com/_/nginx/)  
Official nginx docker image and docs.
- [nginx-resources](https://github.com/fcambus/nginx-resources)  
A collection of resources covering Nginx, Nginx + Lua, OpenResty and Tengine.

So here I'm trying to put together all (my) good patterns and knowledge, and organize it as simply as possible in comparison with complex examples above. So anyone will be able to copy this configs and get a good nginx setup out of the box :)

You can vote for my feature requests in official [docker-nginx](https://github.com/nginxinc/docker-nginx) repo:
* [[Feature Request] Advanced default settings](https://github.com/nginxinc/docker-nginx/issues/593)
* [[Feature Request] Custom envsubst for templating with default values](https://github.com/nginxinc/docker-nginx/issues/592)

Time track:
- [Filipp Frizzy](https://github.com/Friz-zy/) 55.64h

### Configs

#### Main configs
Almost all sections moved from main `nginx.conf` into `conf.d` directory:

* `basic.conf`  
Basic settings, mime types, charset, index, timeouts, open file cache, etc...
* `cache.conf`  
Fastcgi, Proxy and Uwsgi cache setup, see known traps before using ;)
* `gzip.conf`  
Gzip and gzip static
* `log_format.conf`  
Extended log formats
* `real_ip.conf`  
Allow X-Forwarded-For header from local networks and [cloudflare](https://www.cloudflare.com/)
* `request_id.conf`  
Add X-Request-ID header into each request for tracing and debugging
* `security.conf`  
Security settings and headers
* `ssl.conf`  
SSL best practice from [mozilla](https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=intermediate&openssl=1.1.1d&guideline=5.4)

#### Snippets
Templates and includes. You can also use [config generator](https://www.digitalocean.com/community/tools/nginx) from digitalocean :)

* `corps.include.template`  
Template of corps politic for multiple subdomains setup
* `default.conf`  
Example of default config with nginx_status, let's encrypt check and redirect to https
* `fastcgi.include`  
Include for php locations: fastcgi parameters, timeouts and cache example
* `headers.include`  
Include with all headers, see known traps
* `protected_locations.include`  
Include with protected locations with 'deny all'
* `proxy.include`  
Include for proxy locations: proxy headers, parameters, timeouts and cache example
* `referer.include.template`  
Template of referer protection for cases when you concurents use your fail2ban protection against you, see known traps
* `resolver.conf.template`  
Include for dynamic dns resolving, see known traps
* `site.conf.template`  
Template of common site configuration
* `static_location.include`  
Include with location for static files

#### Dockerfile
`Dockerfile` example with build args, configs copying and custom envsubst template engine

#### Docker-compose
`docker-compose.yml` example for nginx

#### Fail2ban
You can use fail2ban for banning some bots even behind load balancer.
`nginx-deny` action will add `deny <ip>;` into `/etc/nginx/conf.d/banned.conf` and reload nginx.

Warning: your evil competitors can use your protection like fail2ban against you, check known traps ;)

Files for copying:
```
fail2ban/jail.local => /etc/fail2ban/jail.local
fail2ban/action-nginx-deny.conf => /etc/fail2ban/action.d/nginx-deny.conf
fail2ban/filter-magento.conf => /etc/fail2ban/filter.d/nginx-magento.conf
fail2ban/filter-wordpress.conf => /etc/fail2ban/filter.d/nginx-wordpress.conf
fail2ban/filter-nginx-noscript.conf => /etc/fail2ban/filter.d/nginx-noscript.conf
```

#### Filebeat
Filebeat by default can't parse extended nginx access log formats, so you should override ingest json:
Copy `filebeat/nginx_access_ingest.json` to `/usr/share/filebeat/module/nginx/access/ingest/default.json`

### Known traps

#### Cache with default settings break all client specific content
If you use fastcgi, proxy or uwsgi cache with default settings like
```
http {

    proxy_cache_path /tmp/cache levels=1:2 keys_zone=mycache:10m max_size=10g 
                inactive=60m use_temp_path=off;

    server {
        listen 80;
        proxy_cache mycache;

        location / {
            proxy_pass http://backend1;
        }

        location /some/path {
            proxy_pass http://backend2;
            proxy_cache_valid any 1m;
            proxy_cache_min_uses 3;
            proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment;
        }
    }
}
```
in both locations Nginx will cache every response. 
So if your site has some login functionality or shopping cart or whatever, 
it will be mixed and most of clients will get response with content of some other clients.

In this configuration I suggest caches only as an additional tool for caching common non 200 status responses:
```
fastcgi_cache_valid 499 500 502 503 504 521 522 523 524 3s; # circuit breaker
fastcgi_cache_valid 404 15m; # cache Not Found for decrease loading to backend
fastcgi_cache_valid 301 308 1h; # cache Permanent Redirect for decrease loading to backend
fastcgi_cache_valid 302 307 5s; # cache Temporary Redirect for decrease loading to backend

# don't cache any other responses
fastcgi_cache_valid 200 0;
fastcgi_cache_valid any 0;
```
And even this one commented out in cache.conf, so you should choose yourself 
and enable it manually for whole site or some locations.

However, how we can safely enable cache for all responses?.
And use cache config like
```
fastcgi_cache_valid 401 0;
fastcgi_cache_valid any 3s;
fastcgi_cache_valid 404 15m;
fastcgi_cache_valid 301 308 1h;
fastcgi_cache_valid 200 5m;
```

1) *The easiest*  
By default, NGINX respects the Cache-Control headers from origin servers. 
It does not cache responses with Cache-Control set to Private, No-Cache, or 
No-Store or with Set-Cookie in the response header. So if your app can add `Cache-Control` 
header into every response - we are done here :) [Example](https://serverfault.com/a/815990)

```
Parameters of caching can also be set directly in the response header. This has higher priority than setting of caching time using the directive.
- The “X-Accel-Expires” header field sets caching time of a response in seconds. The zero value disables caching for a response. If the value starts with the @ prefix, it sets an absolute time in seconds since Epoch, up to which the response may be cached.
- If the header does not include the “X-Accel-Expires” field, parameters of caching may be set in the header fields “Expires” or “Cache-Control”.
- If the header includes the “Set-Cookie” field, such a response will not be cached.
- If the header includes the “Vary” field with the special value “*”, such a response will not be cached (1.7.7). If the header includes the “Vary” field with another value, such a response will be cached taking into account the corresponding request header fields (1.7.7).
Processing of one or more of these response header fields can be disabled using the fastcgi_ignore_headers directive.
```
[ngx_http_fastcgi_module](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)

2) *The most correct*  
If you app can store cache in an external cache database 
like redis or memcached, you can use Nginx 
[redis](https://github.com/openresty/redis2-nginx-module) or 
[memcached](https://nginx.org/en/docs/http/ngx_http_memcached_module.html) 
modules instead of nginx cache for both caching and speeding up your site.

3) *The most difficult*  
You can check URI and cookies by nginx itself, but this is hard 
and add a mess into your configs and risk of mistakes. There is a good example in 
the [engintron](https://github.com/engintron/engintron/blob/master/nginx/proxy_params_dynamic) configs, 
but it's under GPLv2 so I can't include it into my snippets. Also there is a little easier 
[example](https://dev.to/ale_ukr/how-to-make-nginx-cache-cookie-aware-2ffl) how to check only one cookie.

4) *Bonus: the lucky one*  
For static content locations you can just enable cache without any dancing around :)

#### [Adding add_header remove all add_header directives from parent sections](https://www.peterbe.com/plog/be-very-careful-with-your-add_header-in-nginx)

Configuration like
```
add_header Name1 Value1;

location / {
    add_header Name2 Value2;
```
After all produce only `Name2` header in response. 
So use add_header.conf include or copy all headers manually 
into sections under HTTP one.
```
include /etc/nginx/snippets/headers.include
```

#### DNS resolving and cache in Docker, Kubernetes and other dynamic environments

By default, as NGINX starts up or reloads its configuration,
it queries a DNS server to resolve backend dns records.
The DNS server returns the list of backend IPs,
and NGINX uses the default Round Robin algorithm to load balance requests among them.
NGINX chooses the DNS server from the OS configuration file /etc/resolv.conf.
This method is the least flexible way to do service discovery and has the following additional drawbacks:
- If the domain name can’t be resolved, NGINX fails to start or reload its configuration.
- NGINX caches the DNS records until the next restart or configuration reload, ignoring the records’ TTL values.

For dynamic dns resolving in docker, k8s and other dynamic environments,
you should set the Domain Name in a Variable and add resolver directive
to explicitly specify the name server
as NGINX does not refer to /etc/resolv.conf in this case.

```
resolver 127.0.0.1 valid=10s;

server {
    location / {
        set $backend backends.example.com;
        proxy_pass http://$backend;
    }
}
```

You can configure and include `resolver.conf` snippet for manage resolver options:
```
include /etc/nginx/snippets/resolver.conf
```

#### Fail2ban and any other protection can be used against you

Not only that incorrectly configured protection will block valid users,
even right configured protection like fail2ban, especially with `botsearch-common` filter,
can be used for attack to you. For example, you competitors can add to their sites something like
```
<img src="https://{{ your site }}/admin/1.jpg">
<img src="https://{{ your site }}/phpmyadmin/1.jpg">
<img src="https://{{ your site }}/roundcube/1.jpg">
```

Then valid user after visit to the their site will be automatically blocked on your site 😆
You can fight with this practice using `http_referer`, see `snippets/referer.include.template` template ;)
Warning: I have not tested this code yet

#### Default templating engine in official docker image can't proceed variables with default values like `${var:-$DEFAULT}`

By default nginx in docker use [GNU envsubst](https://www.gnu.org/software/gettext/manual/html_node/envsubst-Invocation.html)
that [can't proceed variables with default values](https://stackoverflow.com/questions/50230361/envsubst-default-values-for-unset-variables).
You can use instead [a8m envsubst](https://github.com/a8m/envsubst) or [stephenc envsub](https://github.com/stephenc/envsub),
first one already has a prebuilded binary for x86_64 arch, check the `Dockerfile` in this repo ;)

#### Includes like `<dir>/*.conf` are processed in the alphabetic order

This is important for nginx in docker as all configs are located in one dir

#### Errors like ` failed (24: Too many open files)` or `worker_connections exceed open file resource limit`

Problem with limit of open files (`ulimit -n`)

You can change it
* systemd  
Add into `/etc/systemd/system/nginx.d/override.conf`
```
[Service]
LimitNOFILE=100000
```
* old init system  
Change `/etc/default/nginx`
```
ULIMIT="-n 100000"
```
* docker-compose
```
ulimits:
  nproc: 65535
  nofile:
    soft: 100000
    hard: 100000
```

Maybe you should also change `/etc/security/limits.conf`
```
nginx           hard    nofile          100000
nginx           soft    nofile          100000
www-data        hard    nofile          100000
www-data        soft    nofile          100000
```
and `/etc/sysctl.conf`
```
fs.file-max = 394257
```

### Nginx build info

#### Docker
```
nginx version: nginx/1.17.9
built by gcc 8.3.0 (Debian 8.3.0-6)
built with OpenSSL 1.1.1d  10 Sep 2019
TLS SNI support enabled
configure arguments: 
--prefix=/etc/nginx 
--sbin-path=/usr/sbin/nginx 
--modules-path=/usr/lib/nginx/modules 
--conf-path=/etc/nginx/nginx.conf 
--error-log-path=/var/log/nginx/error.log 
--http-log-path=/var/log/nginx/access.log 
--pid-path=/var/run/nginx.pid 
--lock-path=/var/run/nginx.lock 
--http-client-body-temp-path=/var/cache/nginx/client_temp 
--http-proxy-temp-path=/var/cache/nginx/proxy_temp 
--http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp 
--http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp 
--http-scgi-temp-path=/var/cache/nginx/scgi_temp 
--user=nginx 
--group=nginx 
--with-compat 
--with-file-aio 
--with-threads 
--with-http_addition_module 
--with-http_auth_request_module 
--with-http_dav_module 
--with-http_flv_module 
--with-http_gunzip_module 
--with-http_gzip_static_module 
--with-http_mp4_module 
--with-http_random_index_module 
--with-http_realip_module 
--with-http_secure_link_module 
--with-http_slice_module 
--with-http_ssl_module 
--with-http_stub_status_module 
--with-http_sub_module 
--with-http_v2_module 
--with-mail 
--with-mail_ssl_module 
--with-stream 
--with-stream_realip_module 
--with-stream_ssl_module 
--with-stream_ssl_preread_module 
--with-cc-opt='-g -O2 
-fdebug-prefix-map=/data/builder/debuild/nginx-1.17.9/debian/debuild-base/nginx-1.17.9=. 
-fstack-protector-strong -Wformat -Werror=format-security 
-Wp,-D_FORTIFY_SOURCE=2 -fPIC' 
--with-ld-opt='-Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'
```

#### Ubuntu 18.04 build info
```
nginx version: nginx/1.14.0 (Ubuntu)
built with OpenSSL 1.1.1  11 Sep 2018
TLS SNI support enabled
configure arguments: 
--with-cc-opt='-g -O2 -fdebug-prefix-map=/build/nginx-GkiujU/nginx-1.14.0=. 
-fstack-protector-strong -Wformat -Werror=format-security 
-fPIC -Wdate-time -D_FORTIFY_SOURCE=2' 
--with-ld-opt='-Wl,-Bsymbolic-functions -Wl,-z,relro -Wl,-z,now -fPIC' 
--prefix=/usr/share/nginx 
--conf-path=/etc/nginx/nginx.conf 
--http-log-path=/var/log/nginx/access.log 
--error-log-path=/var/log/nginx/error.log 
--lock-path=/var/lock/nginx.lock 
--pid-path=/run/nginx.pid 
--modules-path=/usr/lib/nginx/modules 
--http-client-body-temp-path=/var/lib/nginx/body 
--http-fastcgi-temp-path=/var/lib/nginx/fastcgi 
--http-proxy-temp-path=/var/lib/nginx/proxy 
--http-scgi-temp-path=/var/lib/nginx/scgi 
--http-uwsgi-temp-path=/var/lib/nginx/uwsgi 
--with-debug 
--with-pcre-jit 
--with-http_ssl_module 
--with-http_stub_status_module 
--with-http_realip_module 
--with-http_auth_request_module 
--with-http_v2_module 
--with-http_dav_module 
--with-http_slice_module 
--with-threads 
--with-http_addition_module 
--with-http_geoip_module=dynamic 
--with-http_gunzip_module 
--with-http_gzip_static_module 
--with-http_image_filter_module=dynamic 
--with-http_sub_module 
--with-http_xslt_module=dynamic 
--with-stream=dynamic 
--with-stream_ssl_module 
--with-mail=dynamic 
--with-mail_ssl_module
```
