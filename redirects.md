# nginx.conf www to non-www and http to https redirects in one server block

This will redirect:
* all requests for http://www.example.com and https://www.example.com to https://example.com
* all requests for http://example.com to https://example.com

```
server {
  server_name example.com www.example.com;
  listen 80;
  listen 443 ssl;

...

  if ($scheme = http) {
     return 301 https://$server_name$request_uri;
  }

  if ($host ~ ^www\.) {
     return 301 https://$server_name$request_uri;
  }

...

}

```

However, it can easily go wrong if you swap server names in `server_name` directive:

```
server {
  server_name www.example.com example.com;

...

  if ($scheme = http) {
     return 301 https://$server_name$request_uri;
  }

  if ($host ~ ^www\.) {
     return 301 https://$server_name$request_uri;
  }

...

}

```

Here is more robust solution:


```
server {
  server_name www.example.com example.com;

...

  if ($scheme = http) {
     return 301 https://$server_name$request_uri;
  }

  if ($host ~ ^www\.) {
     return 301 https://example.com$request_uri;
  }

...

}

```

This feature can be achieved using different server blocks like below, but may break some control panels that does not tolerate big manual changes to config files.

# Redirects using many server blocks

```
server {
  listen 443 ssl;
  server_name example.com;
 
  ...

}

server {
  listen 443 ssl;
  server_name www.example.com;
  return 301 https://example.com$request_uri;

  ...

}

server {
  listen 80;
  server_name example.com www.example.com;
  return 301 https://example.com$request_uri;
 
  ...
  
}
``` 




