# WP Super Cahce nginx.conf example

## Example configuration for Nginx and WordPress with WP Super Cache plugin

```
# redirect www to non-www for https requests
server {
        listen 443 ssl;
        server_name www.example.com;
        ssl_certificate /home/exampleaccount/ssl.cert;
        ssl_certificate_key /home/exampleaccount/ssl.key;
        access_log /var/log/nginx/www.example.com_access_log;
        error_log /var/log/nginx/www.example.com_error_log;
        return 301 https://example.com$request_uri;
       }

# redirect http requests to https
server {
        listen 80;
        server_name example.com www.example.com;
        access_log /var/log/nginx/http_example.com_access_log;
        error_log /var/log/nginx/http_example.com_error_log;
        return 301 https://example.com$request_uri;
       }

server {
        server_name example.com;
        listen 443 ssl;
        ssl_certificate /home/exampleaccount/ssl.cert;
        ssl_certificate_key /home/exampleaccount/ssl.key;
        root /home/exampleaccount/public_html;
        index index.html index.htm index.php;
        access_log /var/log/nginx/example.com_access_log;
        error_log /var/log/nginx/example.com_error_log;

        location = /favicon.ico { return 404; log_not_found off; access_log off; } # remove this line if you have favicon.ico

        # upload big import files (also php.ini configuration needed: upload_max_filesize and post_max_size)
        client_max_body_size 20M;

        # BEGIN GZIP
        gzip on;
        gzip_disable "msie6";
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/html text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript;
        # END GZIP

        set $cache_uri $request_uri;

        # POST requests should always go to PHP
        if ($request_method = POST) {
            set $cache_uri 'null cache';
        }

        # urls with a query string should always go to PHP
        if ($query_string != "") {
            set $cache_uri 'null cache';
        }

        set $TestingThisFix 'Not our WordPress'; # debug

        # Request from WordPress should always go to PHP, otherwise preload will not work
        # Don't forget to change example\.com with your-domain-name\.com and do not delete the '\' before the dot.
        if ($http_user_agent ~* ^WordPress.*\ example\.com$ ) {
            set $cache_uri 'null cache';
            set $TestingThisFix 'Our WordPress'; # debug
        }

        add_header Testing-This-Fix $TestingThisFix;

        # Don't cache uris containing the following segments
        if ($request_uri ~* "(/wp-admin/|/xmlrpc.php|/wp-(app|cron|login|register|mail).php
                              |wp-.*.php|/feed/|index.php|wp-comments-popup.php
                              |wp-links-opml.php|wp-locations.php |sitemap(_index)?.xml
                              |[a-z0-9_-]+-sitemap([0-9]+)?.xml)") {
            set $cache_uri 'null cache';
        }

        # Don't use the cache for logged-in users or recent commenters
        if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+
                             |wp-postpass|wordpress_logged_in") {
            set $cache_uri 'null cache';
        }

        # Set the cache file (we assume that this server block serves only https requests!)
        set $cachefile "/wp-content/cache/supercache/$http_host$cache_uri/index-https.html"

        # Add cache file debug info as header
        add_header X-Cache-File $cachefile;

        # Try in the following order: (1) cachefile, (2) normal url, (3) php
        location / {
            try_files $cachefile $uri $uri/ /index.php;
        }

        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_pass unix:/var/php-nginx/148519495327014.sock/socket;
            fastcgi_param GATEWAY_INTERFACE CGI/1.1;
            fastcgi_param SERVER_SOFTWARE nginx;
            fastcgi_param QUERY_STRING $query_string;
            fastcgi_param REQUEST_METHOD $request_method;
            fastcgi_param CONTENT_TYPE $content_type;
            fastcgi_param CONTENT_LENGTH $content_length;
            fastcgi_param SCRIPT_FILENAME /home/exampleaccount/public_html$fastcgi_script_name;
            fastcgi_param SCRIPT_NAME $fastcgi_script_name;
            fastcgi_param REQUEST_URI $request_uri;
            fastcgi_param DOCUMENT_URI $document_uri;
            fastcgi_param DOCUMENT_ROOT /home/exampleaccount/public_html;
            fastcgi_param SERVER_PROTOCOL $server_protocol;
            fastcgi_param REMOTE_ADDR $remote_addr;
            fastcgi_param REMOTE_PORT $remote_port;
            fastcgi_param SERVER_ADDR $server_addr;
            fastcgi_param SERVER_PORT $server_port;
            fastcgi_param SERVER_NAME $server_name;
            fastcgi_param HTTPS $https;
        }

       }
```

If your server block serves only http requests you should change this line:

```
    set $cachefile "/wp-content/cache/supercache/$http_host$cache_uri/index-https.html"
```

to this:

```
        set $cachefile "/wp-content/cache/supercache/$http_host$cache_uri/index.html";
```

If your server block serves http and https requests you should change it like this:

```
    set $cachefile "/wp-content/cache/supercache/$http_host$cache_uri/index.html";
    if ($scheme = https) {
        set $cachefile "/wp-content/cache/supercache/$http_host$cache_uri/index-https.html";
    }
```

In some guides you will find such example:
```
    # not very good example, however it works
    set $cachefile "/wp-content/cache/supercache/$http_host/$cache_uri/index.html";
    if ($https ~* "on") {
        set $cachefile "/wp-content/cache/supercache/$http_host/$cache_uri/index-https.html";
    }
```

But the symbol `/` before `$cache_uri` is redundant - the `$cache_uri` variable contains leading `/`.

Also, the `$https ~* "on"` is not as efficient as `$scheme = https`.

If possible, avoid using `if`s.


If you want to make an exception only for logged in users, change this:

```
        # Don't use the cache for logged-in users or recent commenters
        if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+
                             |wp-postpass|wordpress_logged_in") {
            set $cache_uri 'null cache';
        }
```

to this:

```
        # Don't use the cache for logged-in users
        if ($http_cookie ~* "wordpress_logged_in") {
            set $cache_uri 'null cache';
        }
```

According to my observation, recent commenters will view their own comments immediately after they post them (the cache is rebuilding on the fly), so you may configure the server to serve cached content to them.

There is also an option on WP Super Cache called "Make known users anonymous so they’re served supercached static files". This way logged in users will receive cached pages, but not directly by Nginx (cache will be server after invoking WordPress).

# WP_CRON

Because the WordPress will be invoked not often (because of Nginx serving cache most of the time), I suggest to disable WP-CRON by adding `define('DISABLE_WP_CRON', true);` to the `wp-config.php` and creating a cron job to be executed every minute with the correct username: `cd /home/UserAccountChangeMe/public_html/ ; php -q wp-cron.php`.


