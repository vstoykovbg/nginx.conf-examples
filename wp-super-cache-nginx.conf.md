# WP Super Cache nginx.conf example

## Example configuration for Nginx and WordPress with WP Super Cache plugin

```
# redirect www to non-www for https requests
server {
        listen 443 ssl http2;
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
        listen 443 ssl http2;
        ssl_certificate /home/exampleaccount/ssl.cert;
        ssl_certificate_key /home/exampleaccount/ssl.key;
        root /home/exampleaccount/public_html;
        index index.html index.htm index.php;
        access_log /var/log/nginx/example.com_access_log;
        error_log /var/log/nginx/example.com_error_log;

        # some https settings
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

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
        gzip_types text/html text/css text/x-component application/x-javascript application/javascript text/javascript text/x-js text/richtext image/svg+xml text/plain text/xsd text/xsl text/xml image/bmp application/java application/msword application/vnd.ms-fontobject application/x-msdownload image/x-icon image/webp application/json application/vnd.ms-access application/vnd.ms-project application/x-font-otf application/vnd.ms-opentype application/vnd.oasis.opendocument.database application/vnd.oasis.opendocument.chart application/vnd.oasis.opendocument.formula application/vnd.oasis.opendocument.graphics application/vnd.oasis.opendocument.spreadsheet application/vnd.oasis.opendocument.text audio/ogg application/pdf application/vnd.ms-powerpoint application/x-shockwave-flash image/tiff application/x-font-ttf audio/wav application/vnd.ms-write application/font-woff application/font-woff2 application/vnd.ms-excel application/xml application/xml+rss;
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

        add_header Testing-This-Fix $TestingThisFix; # debug

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
            fastcgi_pass unix:/var/php-nginx/000000000000000.sock/socket;
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

In some guides you may find such example:
```
    # not very good example, however it works
    set $cachefile "/wp-content/cache/supercache/$http_host/$cache_uri/index.html";
    if ($https ~* "on") {
        set $cachefile "/wp-content/cache/supercache/$http_host/$cache_uri/index-https.html";
    }
```

However, the symbol `/` before `$cache_uri` is redundant - the `$cache_uri` variable contains leading `/`.

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

## WP_CRON

Because the WordPress will not be invoked often (because of Nginx serving cache most of the time), I suggest to disable WP-CRON by adding `define('DISABLE_WP_CRON', true);` to the `wp-config.php` and to create a cron job to be executed every minute with the correct username: `cd /home/UserAccountChangeMe/public_html/ ; php -q wp-cron.php`.

## CloudFlare and Incapsula

Cloudflare/Incapsula may block some or all requests made by WordPress. This may interfere with the *preload*, because *preload* is working by *WP Super Cache* sending requests to the web server using WP's [http API](https://codex.wordpress.org/HTTP_API). These requests are with user agent `WordPress/4.7.2; https://example.com`.

Even if there are no blocked requests, it is not very inefficient to send request to CloudFlare/Incapsula server, the latter to send request to the Nginx, Nginx to invoke WordPress, etc.

The solution is to add your domain name to `/etc/hosts` with your local IP address. For example:

```
93.184.216.34 example.com www.example.com
```

You should remember to change this if you change the IP address where Nginx is listening.

# Using self-signed TLS certificate (a.k.a. SSL certificate)

If you use self-signed TLS certificate preload will not work. In order to make it work you should use a valid TLScertificate or add your TLS certificate to the WordPress's `ca-bundle.crt`:

```
# su myusername
$ cd ~/public_html/wp-includes/certificates
$ cp ca-bundle.crt  ca-bundle.crt.bak
$ cat ~/ssl.cert >> ca-bundle.crt
```

This file will be overwritten when you update your WordPress, so this is not a good permanent solution.

## favicon.ico and robots.txt

If you don't have `/favicon.ico` the WordPress will be invoked every time when someone is visiting your webiste just to say "there is no such file", which is not efficient. Instead, create such file or use configuration like this:

```
location = /favicon.ico { return 404; log_not_found off; access_log off; } # remove this line if you have favicon.ico
```

If you don't have `robots.txt` file the problem is similar – every time when a robot (i.e. Google bot) is visiting your website, WordPress will be invoked just to say "error 404". The simplest and fastest solution is just to create empty `robots.txt` file.

## Serving static .gz files, compressed in advance by WP Super Cache

You may add `gzip_static on;` in the `location` block, but this feature will work only if it is enabled during compilation with `--with-http_gzip_static_module`.

Find this code:

```
        # Try in the following order: (1) cachefile, (2) normal url, (3) php
        location / {
            try_files $cachefile $uri $uri/ /index.php;
        }
```

And change it like this:

```
        # Try in the following order: (1) cachefile, (2) normal url, (3) php
        location / {
            gzip_static on; # Enable Nginx's gzip static module
            try_files $cachefile $uri $uri/ /index.php;
        }
```

To verify that this feature is enabled during compilation:

```
$ nginx -V
nginx version: nginx/1.10.2
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-4) (GCC) 
built with OpenSSL 1.0.1e-fips 11 Feb 2013
TLS SNI support enabled
configure arguments: --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/run/nginx.pid --lock-path=/run/lock/subsys/nginx --user=nginx --group=nginx --with-file-aio --with-ipv6 --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_addition_module --with-http_xslt_module=dynamic --with-http_image_filter_module=dynamic --with-http_geoip_module=dynamic --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-http_perl_module=dynamic --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-google_perftools_module --with-debug --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -m64 -mtune=generic' --with-ld-opt='-Wl,-z,relro -specs=/usr/lib/rpm/redhat/redhat-hardened-ld -Wl,-E'
```

In the example above it is enabled.

To enable creation of `.gz` files by WP Super Cache, go to Settings -> WP Super Cache -> Advanced and check this checkbox:

```
[x] Compress pages so they’re served more quickly to visitors. (Recommended)
```

You may use this shell script to compress all `.css`, `.js`, `.ttp`, `.eot`, `.woff` and `.ttf` files found *in current directory* and its subdirectories:

```
#!/bin/bash
# compress_with_gzip_recursively.bash

exit; # remove this line after you are sure what you are doing

find -name '*.css' -o -name '*.js' -o -name '*.ttp' -o -name '*.eot' -o -name '*.woff' -o -name '*.ttf'  | while read LN; do

gzip --best --verbose --verbose -c -- "$LN" > "$LN.gz"

done
```

This will work also with old gzip versions, which don't support the `--keep` option (to keep the original file after compressing).

You need to run this script again after update.

```
$ cd ~/public_html/
$ ~/bin/compress_with_gzip_recursively.bash
```

Here is what happened when I started this script on my WordPress test install:

```
$ cd ~/public_html/
$ ~/bin/compress_with_gzip_recursively.bash 
./wp-admin/js/editor.js:	 72.6%
./wp-admin/js/user-profile.min.js:	 64.4%
./wp-admin/js/word-count.min.js:	 57.6%
./wp-admin/js/tags-suggest.js:	 61.0%
./wp-admin/js/tags-box.js:	 64.9%
./wp-admin/js/image-edit.js:	 71.2%
./wp-admin/js/updates.min.js:	 79.4%
./wp-admin/js/postbox.min.js:	 63.3%
./wp-admin/js/nav-menu.min.js:	 70.5%
./wp-admin/js/theme.min.js:	 72.3%
./wp-admin/js/custom-header.js:	 52.9%
./wp-admin/js/press-this.min.js:	 62.1%
./wp-admin/js/media-upload.js:	 59.4%
./wp-admin/js/color-picker.js:	 68.7%
./wp-admin/js/iris.min.js:	 65.9%
./wp-admin/js/farbtastic.js:	 67.5%
./wp-admin/js/comment.js:	 62.1%
./wp-admin/js/common.js:	 71.1%
./wp-admin/js/inline-edit-tax.min.js:	 58.9%
./wp-admin/js/media.min.js:	 56.8%
./wp-admin/js/wp-fullscreen-stub.min.js:	 39.0%
./wp-admin/js/edit-comments.js:	 72.7%
./wp-admin/js/plugin-install.min.js:	 57.7%
./wp-admin/js/xfn.js:	 51.1%
./wp-admin/js/set-post-thumbnail.js:	 50.7%
./wp-admin/js/word-count.js:	 69.9%
./wp-admin/js/inline-edit-post.js:	 67.1%
./wp-admin/js/accordion.js:	 63.0%
./wp-admin/js/link.min.js:	 57.4%
./wp-admin/js/post.min.js:	 68.1%
./wp-admin/js/inline-edit-post.min.js:	 64.2%
./wp-admin/js/bookmarklet.js:	 65.8%
./wp-admin/js/set-post-thumbnail.min.js:	 45.7%
./wp-admin/js/media-gallery.min.js:	 42.5%
./wp-admin/js/dashboard.min.js:	 58.2%
./wp-admin/js/postbox.js:	 70.8%
./wp-admin/js/tags.js:	 58.6%
./wp-admin/js/dashboard.js:	 62.0%
./wp-admin/js/link.js:	 60.4%
./wp-admin/js/tags.min.js:	 53.0%
./wp-admin/js/customize-controls.js:	 77.2%
./wp-admin/js/comment.min.js:	 52.6%
./wp-admin/js/user-suggest.js:	 57.8%
./wp-admin/js/common.min.js:	 66.8%
./wp-admin/js/editor-expand.js:	 76.1%
./wp-admin/js/updates.js:	 81.9%
./wp-admin/js/revisions.min.js:	 73.0%
./wp-admin/js/user-profile.js:	 71.3%
./wp-admin/js/media.js:	 62.8%
./wp-admin/js/image-edit.min.js:	 65.8%
./wp-admin/js/widgets.min.js:	 70.3%
./wp-admin/js/accordion.min.js:	 55.6%
./wp-admin/js/wp-fullscreen-stub.js:	 49.7%
./wp-admin/js/revisions.js:	 74.8%
./wp-admin/js/plugin-install.js:	 63.3%
./wp-admin/js/bookmarklet.min.js:	 51.5%
./wp-admin/js/post.js:	 70.6%
./wp-admin/js/customize-nav-menus.min.js:	 74.6%
./wp-admin/js/custom-background.min.js:	 57.1%
./wp-admin/js/custom-background.js:	 61.4%
./wp-admin/js/gallery.js:	 67.2%
./wp-admin/js/widgets.js:	 72.7%
./wp-admin/js/language-chooser.min.js:	 40.6%
./wp-admin/js/svg-painter.min.js:	 47.9%
./wp-admin/js/customize-widgets.min.js:	 71.3%
./wp-admin/js/theme.js:	 74.5%
./wp-admin/js/password-strength-meter.js:	 58.7%
./wp-admin/js/editor-expand.min.js:	 66.0%
./wp-admin/js/svg-painter.js:	 62.2%
./wp-admin/js/customize-nav-menus.js:	 78.5%
./wp-admin/js/color-picker.min.js:	 63.9%
./wp-admin/js/language-chooser.js:	 47.2%
./wp-admin/js/gallery.min.js:	 62.5%
./wp-admin/js/media-upload.min.js:	 50.7%
./wp-admin/js/tags-box.min.js:	 58.2%
./wp-admin/js/password-strength-meter.min.js:	 41.6%
./wp-admin/js/editor.min.js:	 66.5%
./wp-admin/js/media-gallery.js:	 49.7%
./wp-admin/js/user-suggest.min.js:	 51.0%
./wp-admin/js/nav-menu.js:	 73.6%
./wp-admin/js/edit-comments.min.js:	 66.2%
./wp-admin/js/customize-controls.min.js:	 73.8%
./wp-admin/js/customize-widgets.js:	 75.4%
./wp-admin/js/inline-edit-tax.js:	 67.8%
./wp-admin/js/tags-suggest.min.js:	 54.1%
./wp-admin/js/press-this.js:	 72.3%
./wp-admin/js/xfn.min.js:	 42.9%
./wp-admin/css/l10n.min.css:	 73.5%
./wp-admin/css/press-this-editor.min.css:	 44.8%
./wp-admin/css/ie.css:	 74.0%
./wp-admin/css/color-picker-rtl.min.css:	 67.7%
./wp-admin/css/wp-admin.min.css:	 70.3%
./wp-admin/css/edit.min.css:	 75.9%
./wp-admin/css/dashboard-rtl.min.css:	 78.9%
./wp-admin/css/customize-nav-menus-rtl.css:	 80.3%
./wp-admin/css/ie-rtl.min.css:	 71.8%
./wp-admin/css/install.css:	 70.4%
./wp-admin/css/dashboard-rtl.css:	 78.9%
./wp-admin/css/farbtastic.css:	 60.7%
./wp-admin/css/install-rtl.min.css:	 67.0%
./wp-admin/css/install.min.css:	 67.0%
./wp-admin/css/list-tables.min.css:	 80.4%
./wp-admin/css/edit.css:	 77.9%
./wp-admin/css/themes.min.css:	 81.6%
./wp-admin/css/nav-menus.min.css:	 75.7%
./wp-admin/css/edit-rtl.min.css:	 76.0%
./wp-admin/css/wp-admin-rtl.min.css:	 72.7%
./wp-admin/css/wp-admin-rtl.css:	 70.5%
./wp-admin/css/widgets-rtl.css:	 77.7%
./wp-admin/css/forms-rtl.min.css:	 75.8%
./wp-admin/css/dashboard.min.css:	 78.9%
./wp-admin/css/site-icon-rtl.min.css:	 61.0%
./wp-admin/css/press-this-rtl.css:	 79.1%
./wp-admin/css/press-this-rtl.min.css:	 77.7%
./wp-admin/css/press-this.min.css:	 77.7%
./wp-admin/css/widgets.css:	 77.7%
./wp-admin/css/themes-rtl.min.css:	 81.6%
./wp-admin/css/deprecated-media-rtl.min.css:	 69.2%
./wp-admin/css/press-this.css:	 79.1%
./wp-admin/css/about.min.css:	 72.0%
./wp-admin/css/customize-controls.css:	 82.4%
./wp-admin/css/nav-menus.css:	 76.5%
./wp-admin/css/customize-controls-rtl.min.css:	 82.5%
./wp-admin/css/farbtastic-rtl.css:	 60.9%
./wp-admin/css/edit-rtl.css:	 77.9%
./wp-admin/css/colors/blue/colors-rtl.css:	 82.2%
./wp-admin/css/colors/blue/colors.min.css:	 82.5%
./wp-admin/css/colors/blue/colors-rtl.min.css:	 82.5%
./wp-admin/css/colors/blue/colors.css:	 82.2%
./wp-admin/css/colors/ectoplasm/colors-rtl.css:	 82.2%
./wp-admin/css/colors/ectoplasm/colors.min.css:	 82.5%
./wp-admin/css/colors/ectoplasm/colors-rtl.min.css:	 82.5%
./wp-admin/css/colors/ectoplasm/colors.css:	 82.2%
./wp-admin/css/colors/sunrise/colors-rtl.css:	 82.2%
./wp-admin/css/colors/sunrise/colors.min.css:	 82.5%
./wp-admin/css/colors/sunrise/colors-rtl.min.css:	 82.5%
./wp-admin/css/colors/sunrise/colors.css:	 82.2%
./wp-admin/css/colors/midnight/colors-rtl.css:	 82.2%
./wp-admin/css/colors/midnight/colors.min.css:	 82.5%
./wp-admin/css/colors/midnight/colors-rtl.min.css:	 82.5%
./wp-admin/css/colors/midnight/colors.css:	 82.2%
./wp-admin/css/colors/light/colors-rtl.css:	 82.3%
./wp-admin/css/colors/light/colors.min.css:	 82.6%
./wp-admin/css/colors/light/colors-rtl.min.css:	 82.6%
./wp-admin/css/colors/light/colors.css:	 82.3%
./wp-admin/css/colors/ocean/colors-rtl.css:	 82.2%
./wp-admin/css/colors/ocean/colors.min.css:	 82.5%
./wp-admin/css/colors/ocean/colors-rtl.min.css:	 82.5%
./wp-admin/css/colors/ocean/colors.css:	 82.2%
./wp-admin/css/colors/coffee/colors-rtl.css:	 82.2%
./wp-admin/css/colors/coffee/colors.min.css:	 82.5%
./wp-admin/css/colors/coffee/colors-rtl.min.css:	 82.5%
./wp-admin/css/colors/coffee/colors.css:	 82.2%
./wp-admin/css/deprecated-media-rtl.css:	 71.4%
./wp-admin/css/l10n-rtl.min.css:	 73.6%
./wp-admin/css/farbtastic.min.css:	 56.6%
./wp-admin/css/about.css:	 75.7%
./wp-admin/css/admin-menu-rtl.min.css:	 81.5%
./wp-admin/css/color-picker.css:	 70.0%
./wp-admin/css/nav-menus-rtl.css:	 76.5%
./wp-admin/css/press-this-editor-rtl.min.css:	 44.8%
./wp-admin/css/revisions.css:	 76.2%
./wp-admin/css/press-this-editor.css:	 52.8%
./wp-admin/css/l10n-rtl.css:	 71.7%
./wp-admin/css/login.min.css:	 76.1%
./wp-admin/css/list-tables.css:	 80.4%
./wp-admin/css/customize-nav-menus-rtl.min.css:	 81.4%
./wp-admin/css/media-rtl.min.css:	 78.3%
./wp-admin/css/color-picker-rtl.css:	 70.1%
./wp-admin/css/dashboard.css:	 78.9%
./wp-admin/css/revisions-rtl.min.css:	 73.7%
./wp-admin/css/forms.min.css:	 75.8%
./wp-admin/css/site-icon.css:	 66.6%
./wp-admin/css/forms.css:	 78.3%
./wp-admin/css/deprecated-media.css:	 71.4%
./wp-admin/css/common.css:	 78.8%
./wp-admin/css/widgets.min.css:	 76.9%
./wp-admin/css/admin-menu.css:	 80.4%
./wp-admin/css/ie.min.css:	 71.7%
./wp-admin/css/media-rtl.css:	 79.0%
./wp-admin/css/admin-menu.min.css:	 81.5%
./wp-admin/css/about-rtl.min.css:	 72.0%
./wp-admin/css/farbtastic-rtl.min.css:	 56.7%
./wp-admin/css/ie-rtl.css:	 74.0%
./wp-admin/css/admin-menu-rtl.css:	 80.4%
./wp-admin/css/common-rtl.css:	 78.8%
./wp-admin/css/customize-widgets.min.css:	 79.7%
./wp-admin/css/press-this-editor-rtl.css:	 52.8%
./wp-admin/css/list-tables-rtl.css:	 80.4%
./wp-admin/css/revisions-rtl.css:	 76.2%
./wp-admin/css/login-rtl.min.css:	 76.1%
./wp-admin/css/login-rtl.css:	 67.0%
./wp-admin/css/nav-menus-rtl.min.css:	 75.7%
./wp-admin/css/wp-admin.css:	 66.3%
./wp-admin/css/forms-rtl.css:	 78.3%
./wp-admin/css/customize-widgets-rtl.min.css:	 79.7%
./wp-admin/css/l10n.css:	 71.6%
./wp-admin/css/customize-nav-menus.css:	 80.3%
./wp-admin/css/customize-widgets-rtl.css:	 78.8%
./wp-admin/css/common.min.css:	 78.1%
./wp-admin/css/list-tables-rtl.min.css:	 80.4%
./wp-admin/css/site-icon.min.css:	 60.6%
./wp-admin/css/themes-rtl.css:	 81.4%
./wp-admin/css/customize-controls.min.css:	 82.5%
./wp-admin/css/customize-widgets.css:	 78.8%
./wp-admin/css/customize-nav-menus.min.css:	 81.4%
./wp-admin/css/themes.css:	 81.3%
./wp-admin/css/media.min.css:	 78.3%
./wp-admin/css/common-rtl.min.css:	 78.1%
./wp-admin/css/customize-controls-rtl.css:	 82.4%
./wp-admin/css/install-rtl.css:	 70.4%
./wp-admin/css/media.css:	 79.0%
./wp-admin/css/login.css:	 66.9%
./wp-admin/css/site-icon-rtl.css:	 66.7%
./wp-admin/css/deprecated-media.min.css:	 69.2%
./wp-admin/css/color-picker.min.css:	 67.7%
./wp-admin/css/about-rtl.css:	 75.7%
./wp-admin/css/widgets-rtl.min.css:	 76.9%
./wp-admin/css/revisions.min.css:	 73.6%
./wp-content/themes/twentyfifteen/js/color-scheme-control.js:	 71.1%
./wp-content/themes/twentyfifteen/js/customize-preview.js:	 56.5%
./wp-content/themes/twentyfifteen/js/keyboard-image-navigation.js:	 42.9%
./wp-content/themes/twentyfifteen/js/functions.js:	 68.9%
./wp-content/themes/twentyfifteen/js/html5.js:	 49.4%
./wp-content/themes/twentyfifteen/js/skip-link-focus-fix.js:	 41.3%
./wp-content/themes/twentyfifteen/css/ie.css:	 77.5%
./wp-content/themes/twentyfifteen/css/editor-style.css:	 70.9%
./wp-content/themes/twentyfifteen/css/ie7.css:	 64.7%
./wp-content/themes/twentyfifteen/genericons/Genericons.ttf:	 37.0%
./wp-content/themes/twentyfifteen/genericons/genericons.css:	 40.0%
./wp-content/themes/twentyfifteen/genericons/Genericons.eot:	 37.2%
./wp-content/themes/twentyfifteen/genericons/Genericons.woff:	  0.8%
./wp-content/themes/twentyfifteen/style.css:	 85.9%
./wp-content/themes/twentyfifteen/rtl.css:	 79.2%
./wp-content/themes/twentysixteen/js/color-scheme-control.js:	 71.7%
./wp-content/themes/twentysixteen/js/customize-preview.js:	 58.9%
./wp-content/themes/twentysixteen/js/keyboard-image-navigation.js:	 45.0%
./wp-content/themes/twentysixteen/js/functions.js:	 71.5%
./wp-content/themes/twentysixteen/js/html5.js:	 70.0%
./wp-content/themes/twentysixteen/js/skip-link-focus-fix.js:	 46.2%
./wp-content/themes/twentysixteen/css/ie.css:	 57.9%
./wp-content/themes/twentysixteen/css/ie8.css:	 68.4%
./wp-content/themes/twentysixteen/css/editor-style.css:	 71.4%
./wp-content/themes/twentysixteen/css/ie7.css:	 71.4%
./wp-content/themes/twentysixteen/genericons/Genericons.ttf:	 37.6%
./wp-content/themes/twentysixteen/genericons/genericons.css:	 42.1%
./wp-content/themes/twentysixteen/genericons/Genericons.eot:	 37.8%
./wp-content/themes/twentysixteen/genericons/Genericons.woff:	  0.9%
./wp-content/themes/twentysixteen/rtl.css:	 78.3%
./wp-content/themes/twentysixteen/style.css:	 81.4%
./wp-content/themes/twentysixteen/test.css:	 81.4%
./wp-content/themes/twentyseventeen/assets/js/customize-preview.js:	 66.4%
./wp-content/themes/twentyseventeen/assets/js/global.js:	 66.3%
./wp-content/themes/twentyseventeen/assets/js/customize-controls.js:	 56.6%
./wp-content/themes/twentyseventeen/assets/js/jquery.scrollTo.js:	 59.0%
./wp-content/themes/twentyseventeen/assets/js/navigation.js:	 69.5%
./wp-content/themes/twentyseventeen/assets/js/html5.js:	 70.0%
./wp-content/themes/twentyseventeen/assets/js/skip-link-focus-fix.js:	 41.7%
./wp-content/themes/twentyseventeen/assets/css/ie8.css:	 67.8%
./wp-content/themes/twentyseventeen/assets/css/ie9.css:	 78.5%
./wp-content/themes/twentyseventeen/assets/css/colors-dark.css:	 87.4%
./wp-content/themes/twentyseventeen/assets/css/editor-style.css:	 74.6%
./wp-content/themes/twentyseventeen/style.css:	 81.5%
./wp-content/themes/twentyseventeen/rtl.css:	 77.1%
./wp-content/plugins/akismet/_inc/akismet.css:	 74.0%
./wp-content/plugins/akismet/_inc/akismet.js:	 64.2%
./wp-content/plugins/akismet/_inc/form.js:	 57.1%
./wp-includes/js/media-audiovideo.min.js:	 72.9%
./wp-includes/js/customize-preview.min.js:	 70.5%
./wp-includes/js/tw-sack.js:	 69.3%
./wp-includes/js/media-views.min.js:	 77.4%
./wp-includes/js/autosave.min.js:	 60.5%
./wp-includes/js/twemoji.js:	 70.5%
./wp-includes/js/wp-emoji.js:	 61.3%
./wp-includes/js/wp-a11y.js:	 61.6%
./wp-includes/js/underscore.min.js:	 64.6%
./wp-includes/js/twemoji.min.js:	 65.1%
./wp-includes/js/crop/cropper.js:	 69.9%
./wp-includes/js/crop/cropper.css:	 66.9%
./wp-includes/js/wpdialog.js:	 39.1%
./wp-includes/js/utils.js:	 64.2%
./wp-includes/js/customize-preview.js:	 73.4%
./wp-includes/js/wp-embed.js:	 61.3%
./wp-includes/js/wp-pointer.js:	 69.2%
./wp-includes/js/media-editor.min.js:	 66.9%
./wp-includes/js/plupload/handlers.min.js:	 68.1%
./wp-includes/js/plupload/handlers.js:	 69.5%
./wp-includes/js/plupload/wp-plupload.js:	 66.7%
./wp-includes/js/plupload/wp-plupload.min.js:	 60.0%
./wp-includes/js/plupload/plupload.full.min.js:	 67.9%
./wp-includes/js/wp-emoji.min.js:	 50.9%
./wp-includes/js/wp-list-revisions.min.js:	 44.8%
./wp-includes/js/customize-base.min.js:	 69.7%
./wp-includes/js/colorpicker.js:	 71.4%
./wp-includes/js/customize-preview-nav-menus.js:	 72.5%
./wp-includes/js/wp-custom-header.min.js:	 64.4%
./wp-includes/js/imgareaselect/jquery.imgareaselect.min.js:	 62.0%
./wp-includes/js/imgareaselect/imgareaselect.css:	 68.7%
./wp-includes/js/imgareaselect/jquery.imgareaselect.js:	 75.5%
./wp-includes/js/shortcode.js:	 66.8%
./wp-includes/js/wp-lists.js:	 78.8%
./wp-includes/js/wplink.min.js:	 65.0%
./wp-includes/js/thickbox/thickbox.css:	 65.6%
./wp-includes/js/thickbox/thickbox.js:	 69.8%
./wp-includes/js/quicktags.js:	 71.5%
./wp-includes/js/zxcvbn-async.min.js:	 34.0%
./wp-includes/js/utils.min.js:	 56.4%
./wp-includes/js/customize-selective-refresh.min.js:	 66.0%
./wp-includes/js/media-audiovideo.js:	 77.2%
./wp-includes/js/wp-api.js:	 76.3%
./wp-includes/js/json2.min.js:	 57.8%
./wp-includes/js/heartbeat.min.js:	 65.1%
./wp-includes/js/customize-loader.js:	 66.3%
./wp-includes/js/wp-list-revisions.js:	 55.6%
./wp-includes/js/media-grid.js:	 73.7%
./wp-includes/js/wp-util.min.js:	 46.7%
./wp-includes/js/media-models.min.js:	 68.7%
./wp-includes/js/colorpicker.min.js:	 71.1%
./wp-includes/js/wp-auth-check.min.js:	 57.2%
./wp-includes/js/admin-bar.min.js:	 65.9%
./wp-includes/js/wp-ajax-response.min.js:	 54.0%
./wp-includes/js/comment-reply.js:	 60.4%
./wp-includes/js/customize-models.js:	 71.1%
./wp-includes/js/jquery/jquery-migrate.js:	 66.8%
./wp-includes/js/jquery/jquery.form.min.js:	 61.1%
./wp-includes/js/jquery/jquery.masonry.min.js:	 61.6%
./wp-includes/js/jquery/jquery.ui.touch-punch.js:	 52.0%
./wp-includes/js/jquery/jquery.schedule.js:	 71.1%
./wp-includes/js/jquery/jquery.table-hotkeys.min.js:	 63.4%
./wp-includes/js/jquery/suggest.js:	 65.4%
./wp-includes/js/jquery/jquery.query.js:	 56.8%
./wp-includes/js/jquery/jquery.table-hotkeys.js:	 70.4%
./wp-includes/js/jquery/jquery.js:	 65.5%
./wp-includes/js/jquery/jquery.hotkeys.min.js:	 49.1%
./wp-includes/js/jquery/jquery.form.js:	 72.1%
./wp-includes/js/jquery/ui/datepicker.min.js:	 70.0%
./wp-includes/js/jquery/ui/dialog.min.js:	 69.8%
./wp-includes/js/jquery/ui/effect-puff.min.js:	 44.6%
./wp-includes/js/jquery/ui/effect-shake.min.js:	 46.4%
./wp-includes/js/jquery/ui/effect-highlight.min.js:	 45.1%
./wp-includes/js/jquery/ui/effect-transfer.min.js:	 43.3%
./wp-includes/js/jquery/ui/progressbar.min.js:	 64.0%
./wp-includes/js/jquery/ui/effect-fade.min.js:	 37.9%
./wp-includes/js/jquery/ui/mouse.min.js:	 68.1%
./wp-includes/js/jquery/ui/slider.min.js:	 72.1%
./wp-includes/js/jquery/ui/effect-scale.min.js:	 48.5%
./wp-includes/js/jquery/ui/droppable.min.js:	 68.9%
./wp-includes/js/jquery/ui/position.min.js:	 61.0%
./wp-includes/js/jquery/ui/draggable.min.js:	 73.7%
./wp-includes/js/jquery/ui/selectmenu.min.js:	 68.0%
./wp-includes/js/jquery/ui/effect.min.js:	 61.2%
./wp-includes/js/jquery/ui/effect-bounce.min.js:	 45.0%
./wp-includes/js/jquery/ui/spinner.min.js:	 67.1%
./wp-includes/js/jquery/ui/effect-slide.min.js:	 43.0%
./wp-includes/js/jquery/ui/accordion.min.js:	 68.7%
./wp-includes/js/jquery/ui/resizable.min.js:	 71.6%
./wp-includes/js/jquery/ui/effect-pulsate.min.js:	 40.5%
./wp-includes/js/jquery/ui/widget.min.js:	 62.7%
./wp-includes/js/jquery/ui/effect-fold.min.js:	 42.8%
./wp-includes/js/jquery/ui/effect-drop.min.js:	 44.6%
./wp-includes/js/jquery/ui/autocomplete.min.js:	 65.6%
./wp-includes/js/jquery/ui/effect-explode.min.js:	 42.3%
./wp-includes/js/jquery/ui/selectable.min.js:	 69.2%
./wp-includes/js/jquery/ui/core.min.js:	 54.9%
./wp-includes/js/jquery/ui/menu.min.js:	 70.6%
./wp-includes/js/jquery/ui/tabs.min.js:	 68.0%
./wp-includes/js/jquery/ui/effect-size.min.js:	 63.7%
./wp-includes/js/jquery/ui/sortable.min.js:	 73.9%
./wp-includes/js/jquery/ui/effect-clip.min.js:	 41.9%
./wp-includes/js/jquery/ui/effect-blind.min.js:	 44.7%
./wp-includes/js/jquery/ui/tooltip.min.js:	 64.3%
./wp-includes/js/jquery/ui/button.min.js:	 71.5%
./wp-includes/js/jquery/jquery-migrate.min.js:	 60.3%
./wp-includes/js/jquery/jquery.hotkeys.js:	 63.9%
./wp-includes/js/jquery/suggest.min.js:	 55.8%
./wp-includes/js/jquery/jquery.color.min.js:	 58.0%
./wp-includes/js/jquery/jquery.serialize-object.js:	 45.7%
./wp-includes/js/wp-pointer.min.js:	 63.6%
./wp-includes/js/customize-views.js:	 70.1%
./wp-includes/js/wp-embed-template.min.js:	 67.9%
./wp-includes/js/media-models.js:	 74.7%
./wp-includes/js/mce-view.js:	 73.2%
./wp-includes/js/media-views.js:	 80.5%
./wp-includes/js/shortcode.min.js:	 57.0%
./wp-includes/js/quicktags.min.js:	 68.2%
./wp-includes/js/hoverIntent.js:	 67.8%
./wp-includes/js/zxcvbn.min.js:	 52.8%
./wp-includes/js/customize-preview-widgets.js:	 75.1%
./wp-includes/js/wp-custom-header.js:	 71.9%
./wp-includes/js/imagesloaded.min.js:	 69.3%
./wp-includes/js/wp-embed-template.js:	 73.7%
./wp-includes/js/wpdialog.min.js:	 30.4%
./wp-includes/js/wp-auth-check.js:	 60.5%
./wp-includes/js/media-editor.js:	 74.7%
./wp-includes/js/wp-a11y.min.js:	 46.1%
./wp-includes/js/jcrop/jquery.Jcrop.min.js:	 62.3%
./wp-includes/js/jcrop/jquery.Jcrop.min.css:	 71.5%
./wp-includes/js/backbone.min.js:	 67.7%
./wp-includes/js/wp-backbone.min.js:	 63.1%
./wp-includes/js/masonry.min.js:	 70.6%
./wp-includes/js/mediaelement/wp-playlist.js:	 70.7%
./wp-includes/js/mediaelement/mediaelementplayer.min.css:	 84.5%
./wp-includes/js/mediaelement/froogaloop.min.js:	 53.4%
./wp-includes/js/mediaelement/wp-mediaelement.js:	 58.0%
./wp-includes/js/mediaelement/mediaelement-and-player.min.js:	 72.4%
./wp-includes/js/mediaelement/wp-playlist.min.js:	 67.6%
./wp-includes/js/mediaelement/wp-mediaelement.min.css:	 72.8%
./wp-includes/js/mediaelement/wp-mediaelement.css:	 75.1%
./wp-includes/js/mediaelement/wp-mediaelement.min.js:	 49.2%
./wp-includes/js/heartbeat.js:	 72.0%
./wp-includes/js/wp-emoji-release.min.js:	 63.3%
./wp-includes/js/admin-bar.js:	 68.4%
./wp-includes/js/wplink.js:	 71.6%
./wp-includes/js/wp-ajax-response.js:	 60.5%
./wp-includes/js/comment-reply.min.js:	 47.0%
./wp-includes/js/swfupload/handlers.min.js:	 69.8%
./wp-includes/js/swfupload/swfupload.js:	 73.6%
./wp-includes/js/swfupload/handlers.js:	 70.4%
./wp-includes/js/swfupload/plugins/swfupload.speed.js:	 76.5%
./wp-includes/js/swfupload/plugins/swfupload.cookies.js:	 56.1%
./wp-includes/js/swfupload/plugins/swfupload.queue.js:	 72.4%
./wp-includes/js/swfupload/plugins/swfupload.swfobject.js:	 66.6%
./wp-includes/js/mce-view.min.js:	 62.1%
./wp-includes/js/customize-models.min.js:	 68.3%
./wp-includes/js/wp-emoji-loader.min.js:	 57.4%
./wp-includes/js/customize-loader.min.js:	 62.2%
./wp-includes/js/hoverIntent.min.js:	 58.7%
./wp-includes/js/tinymce/themes/inlite/theme.min.js:	 62.3%
./wp-includes/js/tinymce/themes/inlite/theme.js:	 76.9%
./wp-includes/js/tinymce/themes/modern/theme.min.js:	 61.5%
./wp-includes/js/tinymce/themes/modern/theme.js:	 72.1%
./wp-includes/js/tinymce/langs/wp-langs-en.js:	 65.2%
./wp-includes/js/tinymce/tinymce.min.js:	 66.5%
./wp-includes/js/tinymce/plugins/lists/plugin.min.js:	 66.0%
./wp-includes/js/tinymce/plugins/lists/plugin.js:	 76.4%
./wp-includes/js/tinymce/plugins/wordpress/plugin.min.js:	 62.0%
./wp-includes/js/tinymce/plugins/wordpress/plugin.js:	 71.3%
./wp-includes/js/tinymce/plugins/wpemoji/plugin.min.js:	 53.1%
./wp-includes/js/tinymce/plugins/wpemoji/plugin.js:	 63.9%
./wp-includes/js/tinymce/plugins/compat3x/css/dialog.css:	 71.2%
./wp-includes/js/tinymce/plugins/compat3x/plugin.min.js:	 58.1%
./wp-includes/js/tinymce/plugins/compat3x/plugin.js:	 68.6%
./wp-includes/js/tinymce/plugins/wpautoresize/plugin.min.js:	 61.9%
./wp-includes/js/tinymce/plugins/wpautoresize/plugin.js:	 66.7%
./wp-includes/js/tinymce/plugins/paste/plugin.min.js:	 59.6%
./wp-includes/js/tinymce/plugins/paste/plugin.js:	 70.7%
./wp-includes/js/tinymce/plugins/colorpicker/plugin.min.js:	 51.2%
./wp-includes/js/tinymce/plugins/colorpicker/plugin.js:	 62.8%
./wp-includes/js/tinymce/plugins/textcolor/plugin.min.js:	 58.5%
./wp-includes/js/tinymce/plugins/textcolor/plugin.js:	 66.9%
./wp-includes/js/tinymce/plugins/wpdialogs/plugin.min.js:	 53.1%
./wp-includes/js/tinymce/plugins/wpdialogs/plugin.js:	 58.9%
./wp-includes/js/tinymce/plugins/wpembed/plugin.min.js:	 26.1%
./wp-includes/js/tinymce/plugins/wpembed/plugin.js:	 40.5%
./wp-includes/js/tinymce/plugins/tabfocus/plugin.min.js:	 45.7%
./wp-includes/js/tinymce/plugins/tabfocus/plugin.js:	 57.8%
./wp-includes/js/tinymce/plugins/wplink/plugin.min.js:	 63.1%
./wp-includes/js/tinymce/plugins/wplink/plugin.js:	 70.5%
./wp-includes/js/tinymce/plugins/wptextpattern/plugin.min.js:	 55.5%
./wp-includes/js/tinymce/plugins/wptextpattern/plugin.js:	 68.4%
./wp-includes/js/tinymce/plugins/directionality/plugin.min.js:	 46.1%
./wp-includes/js/tinymce/plugins/directionality/plugin.js:	 55.4%
./wp-includes/js/tinymce/plugins/charmap/plugin.min.js:	 62.5%
./wp-includes/js/tinymce/plugins/charmap/plugin.js:	 68.3%
./wp-includes/js/tinymce/plugins/image/plugin.min.js:	 63.2%
./wp-includes/js/tinymce/plugins/image/plugin.js:	 72.3%
./wp-includes/js/tinymce/plugins/wpeditimage/plugin.min.js:	 65.7%
./wp-includes/js/tinymce/plugins/wpeditimage/plugin.js:	 73.9%
./wp-includes/js/tinymce/plugins/wpview/plugin.min.js:	 57.7%
./wp-includes/js/tinymce/plugins/wpview/plugin.js:	 66.0%
./wp-includes/js/tinymce/plugins/media/plugin.min.js:	 64.4%
./wp-includes/js/tinymce/plugins/media/plugin.js:	 72.7%
./wp-includes/js/tinymce/plugins/fullscreen/plugin.min.js:	 58.5%
./wp-includes/js/tinymce/plugins/fullscreen/plugin.js:	 68.5%
./wp-includes/js/tinymce/plugins/hr/plugin.min.js:	 46.3%
./wp-includes/js/tinymce/plugins/hr/plugin.js:	 50.3%
./wp-includes/js/tinymce/plugins/wpgallery/plugin.min.js:	 55.1%
./wp-includes/js/tinymce/plugins/wpgallery/plugin.js:	 63.1%
./wp-includes/js/tinymce/skins/wordpress/wp-content.css:	 69.1%
./wp-includes/js/tinymce/skins/lightgray/content.inline.min.css:	 62.4%
./wp-includes/js/tinymce/skins/lightgray/content.min.css:	 64.0%
./wp-includes/js/tinymce/skins/lightgray/fonts/tinymce-small.ttf:	 47.5%
./wp-includes/js/tinymce/skins/lightgray/fonts/tinymce.eot:	 48.8%
./wp-includes/js/tinymce/skins/lightgray/fonts/tinymce.ttf:	 48.6%
./wp-includes/js/tinymce/skins/lightgray/fonts/tinymce-small.woff:	 47.7%
./wp-includes/js/tinymce/skins/lightgray/fonts/tinymce-small.eot:	 48.0%
./wp-includes/js/tinymce/skins/lightgray/fonts/tinymce.woff:	 48.7%
./wp-includes/js/tinymce/skins/lightgray/skin.ie7.min.css:	 79.7%
./wp-includes/js/tinymce/skins/lightgray/skin.min.css:	 80.1%
./wp-includes/js/tinymce/tiny_mce_popup.js:	 67.3%
./wp-includes/js/tinymce/utils/validate.js:	 68.3%
./wp-includes/js/tinymce/utils/editable_selects.js:	 60.4%
./wp-includes/js/tinymce/utils/form_utils.js:	 64.4%
./wp-includes/js/tinymce/utils/mctabs.js:	 63.1%
./wp-includes/js/wp-lists.min.js:	 65.6%
./wp-includes/js/zxcvbn-async.js:	 45.8%
./wp-includes/js/wp-api.min.js:	 70.5%
./wp-includes/js/wp-backbone.js:	 71.7%
./wp-includes/js/customize-preview-nav-menus.min.js:	 65.6%
./wp-includes/js/customize-selective-refresh.js:	 73.6%
./wp-includes/js/customize-preview-widgets.min.js:	 66.6%
./wp-includes/js/autosave.js:	 71.6%
./wp-includes/js/wp-embed.min.js:	 47.6%
./wp-includes/js/tw-sack.min.js:	 65.8%
./wp-includes/js/json2.js:	 70.0%
./wp-includes/js/swfobject.js:	 61.6%
./wp-includes/js/customize-views.min.js:	 65.7%
./wp-includes/js/customize-base.js:	 71.6%
./wp-includes/js/wp-util.js:	 62.4%
./wp-includes/js/media-grid.min.js:	 71.4%
./wp-includes/js/wp-emoji-loader.js:	 65.2%
./wp-includes/css/media-views.min.css:	 81.8%
./wp-includes/css/jquery-ui-dialog.min.css:	 68.1%
./wp-includes/css/editor.min.css:	 79.4%
./wp-includes/css/dashicons.min.css:	 38.4%
./wp-includes/css/wp-pointer.min.css:	 72.1%
./wp-includes/css/wp-auth-check.min.css:	 63.4%
./wp-includes/css/buttons.min.css:	 79.1%
./wp-includes/css/editor.css:	 80.9%
./wp-includes/css/wp-embed-template-ie.css:	 48.8%
./wp-includes/css/wp-pointer.css:	 72.8%
./wp-includes/css/media-views.css:	 82.8%
./wp-includes/css/media-views-rtl.css:	 82.9%
./wp-includes/css/customize-preview.min.css:	 82.6%
./wp-includes/css/wp-pointer-rtl.css:	 72.7%
./wp-includes/css/wp-embed-template.css:	 75.2%
./wp-includes/css/editor-rtl.min.css:	 79.4%
./wp-includes/css/wp-auth-check-rtl.min.css:	 63.4%
./wp-includes/css/jquery-ui-dialog-rtl.min.css:	 68.2%
./wp-includes/css/buttons.css:	 77.4%
./wp-includes/css/admin-bar.css:	 80.7%
./wp-includes/css/dashicons.css:	 40.9%
./wp-includes/css/buttons-rtl.min.css:	 79.1%
./wp-includes/css/wp-pointer-rtl.min.css:	 71.9%
./wp-includes/css/customize-preview-rtl.css:	 81.9%
./wp-includes/css/admin-bar-rtl.css:	 80.7%
./wp-includes/css/customize-preview-rtl.min.css:	 82.6%
./wp-includes/css/customize-preview.css:	 81.9%
./wp-includes/css/wp-auth-check-rtl.css:	 66.7%
./wp-includes/css/admin-bar.min.css:	 81.5%
./wp-includes/css/jquery-ui-dialog-rtl.css:	 71.4%
./wp-includes/css/media-views-rtl.min.css:	 81.9%
./wp-includes/css/admin-bar-rtl.min.css:	 81.5%
./wp-includes/css/wp-embed-template-ie.min.css:	 48.1%
./wp-includes/css/editor-rtl.css:	 80.9%
./wp-includes/css/buttons-rtl.css:	 77.4%
./wp-includes/css/wp-embed-template.min.css:	 73.1%
./wp-includes/css/wp-auth-check.css:	 66.8%
./wp-includes/css/jquery-ui-dialog.css:	 71.3%
./wp-includes/fonts/dashicons.woff:	  0.8%
./wp-includes/fonts/dashicons.ttf:	 37.5%
./wp-includes/fonts/dashicons.eot:	  0.4%
```

