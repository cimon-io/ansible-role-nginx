Load-balancer role
=========

An Ansible Role that configures nginx as reverse proxy.

Requirements
----------------

None.

Role Variables
----------------

Available variables are listed below, along with default values (see `defaults/main.yml`):

```
vhosts:
  - example.com
```

Sets names of a virtual server.

```
nginx_group: web
```

Upstream group. Takes server name from `{{ hostvars[host][iinventory_hostname] }}`.

```
basic_auth: false
auth_user: "user"
auth_pass: "password"
```

Enables validation of user name and password using the "HTTP Basic Authentication" protocol.

```
https: true
ssl_path: "/etc/nginx/ssl"
ssl_certificate_path: "{{ ssl_path }}/cert.crt"
ssl_certificate_key_path: "{{ ssl_path }}/cert.key"
```

Enable SSL. If `https` parameter is `true`, SSL certificates will be uploaded to `ssl_path`.

```
letsencrypt: false
```

Install Let's Encrypt SSL certificates (standalone mode with stop/start hooks).

```
static_path: "/var/www/static"
```

Sets the `root` directory.

```
applications:
  - application_name: default1
    port: 9998
    location:
      - '/'
      - '/images'
  - application_name: default2
    port: 9999
    location:
      - '~ /bin/*'
```

Add `application_name` upstream. Nginx will redirect requests to backend port `port`, according to `location`.

Dependencies
----------------

Role: nginx.

Example Playbook
----------------

Example:

    - hosts: balancer
      roles:
         - role: load-balancer
           applications:
             - name: "backend"
               port: 8080
               nginx_group: "web"
               location:
                 - "/"
             - name: "frontend"
               port: 8081
               nginx_group: "frontend"
               location:
                 - "= /"
                 - "~ /images"
           vhosts:
             - domain.tld

License
-------

MIT

