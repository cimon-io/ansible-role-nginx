[![Build Status](https://travis-ci.org/cimon-io/ansible-role-nginx.svg?branch=master)](https://travis-ci.org/cimon-io/ansible-role-nginx)

# Ansible Role NGINX

An ansible role that installs and configures the __NGINX__ web server. It also allows you to specify SSL settings and configure any number of sites.

The role includes the following tasks:
1. Install NGINX and all required software specified with the variable `nginx_requirements`.
2. Configure NGINX and generate default SSL.
3. Add the Let's Encrypt or other SSL certificate if necessary.
4. Set up NGINX sites.

This role can be run under all versions of Ubuntu.

## Requirements

None

## Role Variables

Available variables are listed below, along with default values.

### Site configuration

For site configuration variables see `defaults/main.yml`. All sites parameters can be specified by the `nginx_sites` dictionary. The list `nginx_letsencrypt_sites` allows to define which of the sites should get SSL certificates from Let's Encrypt. The variable `nginx_auth_basic` is used to add user credentials to a password file.

```yaml
nginx_sites: {}             # Sites with parameters (see below)
nginx_letsencrypt_sites: [] # List of `nginx_sites` key values for Let's Encrypt SSL
nginx_auth_basic: []        # User name and password for basic authentication
```

The variable `nginx_auth_basic` allows to limit access to resources by validating user name and password using the "HTTP Basic Authentication" protocol. The specified values are saved in the `.htpasswd` file at `etc/nginx`. Each record has the following format:

```yaml
nginx_auth_basic:
  - name: authname
    passwords:
      admin1: pass1  # username:password
      admin2: pass2
```

For the `nginx_sites` items you need to set the configuration name and necessary site parameters for each specific `nginx_sites` key in the following format:

```yaml
nginx_sites:
  webserver:   # A site configuration name
    # Particular site parameters
```

The parameter `server_name` sets one or several names of a specific virtual server. The first [server name](https://nginx.org/en/docs/http/server_names.html) is regarded as a primary one. You can use an asterisk (`*`) to replace the first or the last part of a name. Start a server name with a tilde (`~`) to use a regular expression inside it. It is also possible to specify an empty server name ("") that allows this server to process requests without the "Host" header field.

```yaml
server_name:
  - example.com
```

The `http_listen_port` and `https_listen_port` parameters describe all ports on which the NGINX server should accept requests (for http and https connections, respectively).

```yaml
http_listen_port: 80
https_listen_port: 443
```

The following set of parameters provide the necessary support for https.

```yaml
http2: false         # Enable/disable HTTP/2 connections
https: false         # Enable/disable HTTPS connections
ssl_key_file: ""     # Path to the SSL secret key file
ssl_cert_file: ""    # Path to the SSL certificate file
```

Use the `root` parameter to set up a root directory for requests. A path to the file will be constructed by adding a specific URI to the root value.

```yaml
root: ""              # NGINX root directory for requests
```

The parameter `limit_req_zone` allows to limit the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address. `Leaky bucket` algorithm is used for this. A zone `size` can be specified in bytes (without suffixes), kilobytes (suffixes k and K) or megabytes (suffixes m and M). One megabyte zone can include about 16 thousand 64-byte states or about 8 thousand 128-byte states. If a zone storage is exhausted, the least recently used state is removed. A `rate` is set in requests per second (r/s). If a rate of less than one request per second is desired, it is specified in request per minute (r/m).

```yaml
limit_req_zone: []
# Each list item can contain the following parameters:
- zone: zone1   # A specific zone name
  size: 10m     # Zone size: 10 megabytes
  rate: 30r/s   # Request processing rate cannot exceed 30r/s at an average
```

The parameter `limit_conn_zone` is used to limit the number of connections per the defined key, in particular, the number of connections from a single IP address. A connection is counted only if it has a request processed by the server and the whole request header has already been read. If the zone storage is exhausted, the server will return the error to all further requests.

```yaml
limit_conn_zone: []
# For each zone the following parameters should be specified:
- zone: zone2   # A specific shared memory zone name
  size: 10m     # Define a 10-megabyte zone
```

The parameter `upstreams` is used to define sets of servers. If you are configuring NGINX as a load balancer, you can unite some servers into groups to manage them simultaneously. Such servers can listen on different ports. If an error occurs during communication with a server, the request will be passed to the next server, and so on until all of the functioning servers will be tried.

```yaml
upstreams: []
```

For each [upstream](https://nginx.org/en/docs/http/ngx_http_upstream_module.html) group you are able to specify a `name`, a `port`, a list of servers (particular names `hosts` or/and some group names `group`) and their parameters, `location` values and `proxy_options`.

```yaml
upstreams:
  - name:             # An upstream name
    port:             # Which port to listen to
    hosts: []         # Particular host parameters
    group:            # Group of hosts
    location: []      # Location parameters
    proxy_options: [] # Additional upstream options
```

For each `hosts` item you can define its `name` and a string variable `nginx_upstream_options` which includes:
- `max_fails` - a number of unsuccessful attempts to communicate with the server that should happen during `fail_timeout` interval to consider the server unavailable for a duration which is also set by the `fail_timeout` parameter. Use a zero value to disable accounting of attempts.
- `fail_timeout` - the time during which the specified number of unsuccessful attempts to communicate with the server should happen to consider the server unavailable. It is also used as a period of time during which the server will be considered unavailable after these unsuccessful attempts.

The example of `hosts` variable is given below:

```yaml
hosts:
  - { name: web, nginx_upstream_options: "max_fails=3 fail_timeout=30s" }
```

For an upstream you can set a `group` name instead of (or along with) separate `hosts` values. In this case all hosts of this group (from the Ansible Playbook) will be regarded as this upstream servers. Server names will be taken from `groups[group][inventory_hostname]`.

You are able to configure the following `location` parameters as well:
- `uri` - a request URI with one of the possible modifiers at the beginning, if necessary (`[ = | ~ | ~* | ^~ ]`). Regular expressions are specified with the preceding `~*` modifier (for case-insensitive matching), or the `~` modifier (for case-sensitive matching). If the longest matching prefix location contains the `^~` modifier, regular expressions are not checked. The modifier `=` makes it possible to define an exact match of URI and location.
- `root` - a root directory for NGINX requests.
- `auth` - a string which enables validation of user name and password using the "HTTP Basic Authentication" protocol. Use a special value `off` to cancel the effect of a previous level directive.
- `limit_conn` - the maximum number of simultaneous active connections for a specific shared memory `zone`. Several groups can share the same zone. In this case, it is enough to specify the size only once. The zero `size` value means that there is no limit.
- `limit_req` - sets a shared memory `zone` and a maximum `burst` size of requests. If the rate of incoming requests exceeds the specified value, their processing is delayed so that requests are processed with a given rate. Use an option `nodelay` if extra requests within the burst limit should not be delayed.
- `autoindex` - enables or disables the directory listing output.
- `options` - add necessary options to the location.
- `try_files` - allows to check the existence of files in the specified order and uses the first found file for request processing (the processing is performed in the current location context). To check the directory's existence you should specify a slash at the end of a name. The second "$uri/" value is used for an internal redirect if none of the files were found. The last parameter is an error code.

The upstream parameter `location` can be used in the next way:

```yaml
location:
  - uri: "/"
    root: /var/www/html
  - uri: "/images"
    limit_conn: { zone: zone2, size: 2 }
```

The next upstream parameter `proxy_options` allows to add a list of necessary options to `location @upstream`:

```yaml
proxy_options:
  - 'proxy_set_header Upgrade $http_upgrade'
  - 'proxy_set_header Connection "upgrade"'
```

Some upstream examples are given below along with available options.

```yaml
upstreams:
  - name: default1      # An upstream name
    port: 9997          # A port to listen to
    hosts:              # A list of NGINX virtual server definitions
      - { name: web, upstream_options: "max_fails=3 fail_timeout=30s" }
    location:
      - uri: "/"
        root: /var/www/html
        autoindex: true
  - name: default2
    port: 9998
    location:
      - uri: "~ /blah/*"
        auth: "Basic auth"
  - name: default3
    port: 9999
    location:
      - uri: "= /socket"
        options:
          - 'proxy_set_header Upgrade $http_upgrade'
          - 'proxy_set_header Connection "upgrade"'
        limit_req: { zone: zone1, options: "burst=5 nodelay" }
```

You are able to define [locations](https://nginx.org/en/docs/http/ngx_http_core_module.html#location) with the next site configuration parameter `locations`. It can include the same parameters as a `location` inside the `upstreams` section.

```yaml
locations: []
# Example parameters:
- uri: "/examples"
  root: "/var/www/data"
  try_files: "$uri $uri/ =404"
```

The parameter `keepalive_timeout` sets up NGINX keepalive time. The value should be set to 10s or higher for sites with significant polling-style traffic (for example, AJAX-powered sites). Otherwise, you can set lower values.

```yaml
keepalive_timeout: "65"
```

The parameter `client_max_body_size` is used to specify the maximum allowed size of the client's request body (defined in the "Content-Length" request header field).

```yaml
client_max_body_size: "1m"
```

The parameter `add_header` allows you to add some header fields ("Expires" or "Cache-Control"), and arbitrary fields, to a response header.

```yaml
add_header: []
# Header examples:
  - X-Frame-Options SAMEORIGIN
  - X-Content-Type-Options nosniff
  - X-XSS-Protection "1; mode=block"
```

### Installation

Use `nginx_requirements` to install a list of desired packages. The required ones are already added.

```yaml
nginx_requirements:
  - python-pycurl
  - python-passlib
  - openssl
```

Add NGINX stable repository from PPA and install its signing key with the next variable:

```yaml
nginx_repo: "ppa:nginx/stable"
```

Specify `nginx-extras` package version you need to install:

```yaml
nginx_version: "1.12"
```

Installation pin-priority can be set by the `nginx_apt_pin_priority` parameter. By default, it's 550. All available values are given below:

- __Priority < 0__ - prevent the version from being installed.
- __0 < Priority < 100__ - a version will be installed only if there is no installed version of the package
- __100 <= Priority < 500__ - a version will be installed unless there is a version available belonging to some other distribution or the installed version is more recent.
- __500 <= Priority < 990__ - a required version will be installed unless there is a version available belonging to the target release or the installed version is more recent.
- __990 <= Priority < 1000__ - a version will be installed even if it does not come from the target release, unless the installed version is more recent.
- __Priority >= 1000__ - a version will be installed even if this constitutes a downgrade of the package.

```yaml
nginx_apt_pin_priority: 550
```

A version of `certbot` script that need to be downloaded can be specified with the following variable:

```yaml
nginx_certbot_version: "v0.19.0"
```

Set how often cache should be updated (in seconds):

```yaml
apt_cache_valid_time: 86400  # Update the apt cache if it is older than 1 day
```

### NGINX configuration

The variable `nginx_user` defines user credentials used by worker processes.

```yaml
nginx_user: "www-data"            # A user for whom NGINX is installed
```

The variable `nginx_worker_processes` defines the number of worker processes which NGINX should start. The optimal value depends on the number of CPU cores, the number of hard disk drives that store data, and load level. By default, it is set to "auto" to detect the number of CPU cores automatically. The following parameter `nginx_worker_connections` sets the maximum number of simultaneous connections that can be opened by a worker process. This number includes all connections, not only connections with clients.

```yaml
# The number of worker processes or "auto"
nginx_worker_processes: "auto"
# The maximum number of simultaneous connections for a worker process
nginx_worker_connections: "768"
```

You are able to specify how many connections a worker process will accept at a time with the parameter `nginx_multi_accept`. If it's value is set to "on", a worker process will accept all new connections at a time. Otherwise, the only connection will be accepted at a time.

```yaml
nginx_multi_accept: "off"
```

The following variables are used to enable or disable some additional options.

```yaml
nginx_sendfile: "on"     # Enables or disables the use of sendfile()
nginx_tcp_nopush: "on"   # Enables or disables the use of the TCP_CORK socket option
nginx_tcp_nodelay: "on"  # Enables or disables the use of the TCP_NODELAY option
```

The option `nginx_tcp_nopush` can be enabled only when `nginx_sendfile` is enabled. It allows you to:
- send the response header and the beginning of a file in one packet;
- send a file in full packets.

The option `nginx_tcp_nodelay` can be enabled only when a connection is transitioned into the keep-alive state. To specify a timeout during which a keep-alive client connection will stay open on the server side, use the `nginx_keepalive_timeout` parameter. The "0" value disables keep-alive client connections. The second parameter value is optional and can be used to set the "Keep-Alive: timeout=time" response header field.

```yaml
# A number of seconds during which a client connection won't be closed by server
nginx_keepalive_timeout: "65"
```

Use the following variables to configure hash tables size for the server names and for the types. The size of a table is expressed in buckets. By default, the bucket size for the types is set to 64. The default maximum size of the server names hash tables is 512. The hash bucket size parameter is aligned to the size that is a multiple of the processor's cache line size to speed up key search in a hash by reducing the number of memory accesses.

```yaml
nginx_types_hash_max_size: "2048"          # Sets the maximum size of the types hash tables
nginx_server_names_hash_bucket_size: "64"  # Sets the bucket size for the server names hash tables
```

The following variables are used to configure response compression using the `gzip` method. This often helps to reduce the size of transmitted data by half or even more.

```yaml
nginx_gzip: "on"            # Enables or disables gzipping of responses
nginx_gzip_comp_level: "6"  # Sets a gzip compression level of a response (from 1 to 9)
nginx_gzip_types: "text/plain text/css application/json application/javascript text/xml
                  application/xml application/xml+rss text/javascript"  # Specified MIME types
```

Responses with the "text/html" type are always compressed. Use an asterisk symbol (`*`) to add compression for all MIME types.

The variable `nginx_log_format` specifies log format using the following syntax:

```yaml
nginx_log_format: {}
# A log format includes parameters:
  name: combined
# Default log format
  string: '$remote_addr - $remote_user [$time_local] "$request" $status
          $body_bytes_sent "$http_referer""$http_user_agent"'
```

The log format can contain common variables, and variables that exist only at the time of a log write:
- `$remote_addr` - client address;
- `$remote_user` - user name supplied with the Basic authentication;
- `$request` - full original request line;
- `$http_referer` - the page address (if there is any) that led the user's browser to this page;
- `$http_user_agent` - the browser with which the user requested this page (the User-Agent header field);
- `$bytes_sent` - the total number of bytes sent to a client;
- `$body_bytes_sent` - the number of bytes sent to a client except of headers;
- `$connection` - connection serial number;
- `$connection_requests` - the current number of requests made through a connection;
- `$msec` - time in seconds with a milliseconds resolution at the time of the log write;
- `$pipe` - "p" if request was pipelined, "." otherwise;
- `$request_length` - request length (including request line, header, and request body);
- `$request_time` - request processing time in seconds with a milliseconds resolution; time elapsed between the first bytes were read from the client and the log write after the last bytes were sent to the client;
- `$status` - response status code;
- `$time_iso8601` - local time in the ISO 8601 standard format;
- `$time_local` - local time in the Common Log Format.

### SSL settings

The variable `nginx_ssl_session_timeout` specifies a time during which a client can reuse the session parameters.

```yaml
nginx_ssl_session_timeout: "1d"  # Use session parameters during 1 day
```

The following variable sets types and sizes of caches that store session parameters. A cache can be any of the following types:
- `off` - the usage of a session cache is strictly prohibited (sessions won't be reused).
- `none` - the usage of a session cache is gently disallowed (sessions may be reused, but NGINX does not actually store their parameters in the cache).
- `builtin` - a cache built in OpenSSL; it can be used by one worker process only. The cache size is specified in sessions. If size is not given, it is equal to 20480 sessions. Use of the built-in cache can cause memory fragmentation.
- `shared` - a cache is shared between all worker processes. Its size is specified in bytes (one megabyte can store about 4000 sessions). Each shared cache should have an arbitrary name. A cache with the same name can be used in several virtual servers.

The last two types can be used simultaneously. In this case they should be described in the same line separated with a space. However, using shared cache without the built-in one is more efficient.

```yaml
nginx_ssl_session_cache: "shared:SSL:50m"
```

The following variable enables or disables session resumption through `TLS session tickets`.

```yaml
nginx_ssl_session_tickets: "off"  # Set to "on" to enable session resumption
```

Set SSL protocols separated with a space in `nginx_ssl_protocols`.

```yaml
nginx_ssl_protocols: "TLSv1.2"
```

Next variable specifies the enabled ciphers in the format understood by the OpenSSL library.

```yaml
nginx_ssl_ciphers: "'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:
  ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA38:
  ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256'"

# Specifies that server ciphers should be preferred over client ciphers
# when using the SSLv3 and TLS protocols
nginx_ssl_prefer_server_ciphers: "on"
```

The remaining variables allow to set some SSL files paths. By default, SSL certificate and key paths are generated with `nginx_ssl_path` value and the current `nginx_config_name` file name mentioned above.

```yaml
nginx_ssl_path: "/etc/nginx/ssl"   # Root SSL files path for different configurations
# Default SSL certificate, generated automatically during NGINX configuration
nginx_ssl_default_key_path: "/etc/ssl/private/selfsigned-default.key"
nginx_ssl_default_crt_path: "/etc/ssl/certs/selfsigned-default.crt"
```

## Dependencies

None

## Playbook examples

NGINX basic configurations differ depending on your goals. It can be used as a load balances, for managing static landing pages or backend with various features. Here's some examples of the NGINX role settings.

```yaml
nginx_auth_basic:
  - name: adminpanel
    passwords:
      admin: admin
```

###### Load-balancer

```yaml
nginx_sites:
  load-balancer:
    limit_req_zone:
      - { zone: zone1, size: 10m, rate: 30r/s }
    limit_conn_zone:
      - { zone: zone2, size: 10m }
    add_header:
      - 'X-Frame-Options SAMEORIGIN'
      - 'X-Content-Type-Options nosniff'
      - 'X-XSS-Protection "1; mode=block"'
    upstreams:
      - name: stats
        port: 80
        hosts:
          - { name: "{{ hostvars['stat-service'].inventory_hostname }}" }
        location:
          - uri: '/stats'
            limit_req: { zone: zone1, options: "burst=5" }
        proxy_options:
          - 'proxy_set_header Upgrade $http_upgrade'
          - 'proxy_set_header Connection "upgrade"'
      - name: landing
        port: 80
        hosts:
          - { name: "{{ hostvars['landing'].inventory_hostname }}" }
        location:
          - uri: '= /'
      - name: backend
        port: 80
        group: "backend-servers"
        location:
          - uri: "/"
            limit_conn: { zone: zone2, size: 5 }
    server_name:
      - example.com
```

###### Static landing page

```yaml
nginx_sites:
  landing:
    root: "/var/www/landing"
    server_name:
      - landing.example.com
```

###### Back end

```yaml
nginx_sites:
  stat:
    root: "/var/www/stat"
    upstreams:
      - name: ws_rails
        port: 3000
        hosts:
          - { name: "127.0.0.1" }
        location:
          - uri: "/"
        proxy_options:
          - 'proxy_set_header Upgrade $http_upgrade'
          - 'proxy_set_header Connection "upgrade"'
    server_name:
      - stat.example.com
```

###### Back end with some features

```yaml
nginx_sites:
  backend:
    upstreams:
      - name: rails
        port: 3000
        hosts:
          - { name: "127.0.0.1" }
        location:
          - uri: "/"
          - uri: "/admin"
            auth: "adminpanel"
    locations:
      - uri: "/data"
        autoindex: true
        root: "/var/www/data"
    server_name:
      - backend.example.com
```

## License

Licensed under the [MIT License](https://opensource.org/licenses/MIT).
