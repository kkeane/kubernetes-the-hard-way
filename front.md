# Setting up the Front-end load balancer

Since we are using multiple redundant control-plane servers, we need a
mechanism for selecting one or the other.

For this exercise, we are using a simple nginx-based load balancer. Note
that this is a single point of failure in an otherwise high-availability
system, so in production you may want to implement a more robust approach.

This load balancer is used for two specific use cases:

- accessing the API server from outside the cluster on port 6443
- providing a basic health-monitoring Web page for the cluster servers on
  port 80. This will be publicly available without any authentication required.

The load balancer may also be accessed from within the cluster when
bootstrapping a new node. Once a node has been bootstrapped, it will access the
API server via the service network.

This server does not need any certificates since the encrypted traffic will
simply be passed on to one of the API servers.

## Setup of the load balancer

Install nginx and some prerequisites we need

    yum install nginx libselinux-python policycoreutils-python

Allow nginx to listen on ports 80, 443 and 6443

   semanage port -a -t http_port_t -p tcp 6443
   firewall-cmd --permanent --add-service=http
   firewall-cmd --permanent --add-service=https
   firewall-cmd --permanent --add-port=6443/tcp
   systemctl reload firewalld

Create the nginx configuration file. Replace /etc/nginx/nginx.conf with the
following. This configuration will serve a healthz page at
http://front.vagrant/healthz as well as load-balance all traffic coming in on
port 6443 to one of the control-plane servers.


```
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        rewrite ^/$ $scheme://$host/healthz.html permanent;
        rewrite ^/healthz$ $scheme://$host/healthz.html permanent;

        {% for cp in groups[usd_kubernetes_cp_group] -%}
        location /{{ cp }}/healthz {
            proxy_pass                    https://{{ hostvars[cp].ansible_fqdn }}:{{ usd_kubernetes_api_port }}/healthz;
            proxy_ssl_trusted_certificate {{ usd_ca_cert_directory }}/{{ usd_kubernetes_cluster_name }}-ca.pem;
        }
        {% endfor %}
    }
}

# Access to the API server is with a stream proxy so the API server rather than the
# load balancer will handle the certs.
# Side note: for this to work, the TLS cert on the API server must include the load
# balancer's FQDN!
stream {
  #-----------------------------------------------------------------------------
  # Load balancing the api servers
  upstream k8sapi {
  {% for cp in groups[usd_kubernetes_cp_group] -%}
    server {{ hostvars[cp].ansible_fqdn }}:{{ usd_kubernetes_api_port }};
  {% endfor %}

  }

  server {
    listen 6443;
    proxy_pass k8sapi;
  }
}
```

Create the following file as /usr/share/nginx/html/healthz.html . Of course
you can enhance this page as you like!

```
<!DOCTYPE html>
<html>
<body>

<h1>Kubenetes The Hard Way Control Plane Health Status</h1>
<h2>cp1</h2>
<iframe id="healthz-cp1"
   title="CP1"
   src="/cp1/healthz"
   style="border:none;"></iframe>
<h2>cp2</h2>
<iframe id="healthz-cp2"
   title="CP2"
   src="/cp2/healthz"
   style="border:none;"></iframe>

</body>
</html>

```

Enable and start nginx

    systemctl enable nginx
    systemctl start nginx

## Maintenance

The front-end server is completely stateless, and therefore does not need to be
backed up. If implemented using an automation tool, the easiest disaster
recovery is to rebuild it.

Next: [The worker nodes](./worker.md)

