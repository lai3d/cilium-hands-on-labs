
# (KGLL-212) Kong Gateway Installation using Helm

```bash
KongEduLab:~/KGLL-212$ ll
total 36
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:02 ./
drwxr-xr-x  9 ubuntu ubuntu 4096 Jul  9 09:22 ../
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:20 base/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
KongEduLab:~/KGLL-212$ openssl rand -writerand .rnd
KongEduLab:~/KGLL-212$ ll
total 40
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:26 ./
drwxr-xr-x  9 ubuntu ubuntu 4096 Jul  9 09:22 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:20 base/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
KongEduLab:~/KGLL-212$ openssl req -new -x509 -nodes -newkey ec:<(openssl ecparam -name secp384r1) \
>   -keyout ./cluster.key -out ./cluster.crt \
>   -days 1095 -subj "/CN=kong_clustering"
Generating an EC private key
writing new private key to './cluster.key'
-----
KongEduLab:~/KGLL-212$ ll
total 48
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:26 ./
drwxr-xr-x  9 ubuntu ubuntu 4096 Jul  9 09:22 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:20 base/
-rw-rw-r--  1 ubuntu ubuntu  676 Jul  9 09:26 cluster.crt
-rw-------  1 ubuntu ubuntu  306 Jul  9 09:26 cluster.key
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
KongEduLab:~/KGLL-212$ kubectl create namespace kong
Error from server (AlreadyExists): namespaces "kong" already exists
KongEduLab:~/KGLL-212$  kubectl create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong
secret/kong-cluster-cert created
KongEduLab:~/KGLL-212$ kubectl -n kong create secret tls kong-manager-tls --key="/etc/kong/ssl/server.key" --cert="/etc/kong/ssl/server.crt"
secret/kong-manager-tls created
KongEduLab:~/KGLL-212$ kubectl -n kong create secret tls kong-admin-tls --key="/etc/kong/ssl/server.key" --cert="/etc/kong/ssl/server.crt"
secret/kong-admin-tls created
KongEduLab:~/KGLL-212$ kubectl -n kong create secret tls kong-portal-tls --key="/etc/kong/ssl/server.key" --cert="/etc/kong/ssl/server.crt"
secret/kong-portal-tls created
KongEduLab:~/KGLL-212$ kubectl create secret generic kong-enterprise-license -n kong --from-file=license=/etc/kong/license.json
secret/kong-enterprise-license created
KongEduLab:~/KGLL-212$ cat << EOF > admin_gui_session_conf
> {
>     "cookie_name":"admin_session",
>     "cookie_samesite":"off",
>     "secret":"kong",
>     "cookie_secure":true,
>     "storage":"kong"
> }
> EOF
KongEduLab:~/KGLL-212$ ll
total 52
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:29 ./
drwxr-xr-x  9 ubuntu ubuntu 4096 Jul  9 09:22 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
-rw-rw-r--  1 ubuntu ubuntu  136 Jul  9 09:29 admin_gui_session_conf
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:20 base/
-rw-rw-r--  1 ubuntu ubuntu  676 Jul  9 09:26 cluster.crt
-rw-------  1 ubuntu ubuntu  306 Jul  9 09:26 cluster.key
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
KongEduLab:~/KGLL-212$ kubectl create secret generic kong-session-config -n kong --from-file=admin_gui_session_conf
secret/kong-session-config created
KongEduLab:~/KGLL-212$ kubectl create secret generic kong-enterprise-superuser-password --from-literal=password=password -n kong
secret/kong-enterprise-superuser-password created
KongEduLab:~/KGLL-212$ cat << EOF > portal_gui_session_conf
> {
>     "cookie_name":"portal_session",
>     "cookie_samesite":"off",
>     "secret":"kong",
>     "cookie_secure":true,
>     "cookie_domain":".labs.konghq.com",
>     "storage":"kong"
> }
> EOF
KongEduLab:~/KGLL-212$ ll
total 56
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:30 ./
drwxr-xr-x  9 ubuntu ubuntu 4096 Jul  9 09:22 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
-rw-rw-r--  1 ubuntu ubuntu  136 Jul  9 09:29 admin_gui_session_conf
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:20 base/
-rw-rw-r--  1 ubuntu ubuntu  676 Jul  9 09:26 cluster.crt
-rw-------  1 ubuntu ubuntu  306 Jul  9 09:26 cluster.key
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
-rw-rw-r--  1 ubuntu ubuntu  177 Jul  9 09:30 portal_gui_session_conf
KongEduLab:~/KGLL-212$ kubectl create secret generic kong-portal-session-config -n kong --from-file=portal_session_conf=portal_gui_session_conf
secret/kong-portal-session-config created
KongEduLab:~/KGLL-212$ kubectl create namespace kong-dp
namespace/kong-dp created
KongEduLab:~/KGLL-212$ kubectl create secret tls kong-cluster-cert --cert=./cluster.crt --key=./cluster.key -n kong-dp
secret/kong-cluster-cert created
KongEduLab:~/KGLL-212$ kubectl create secret generic kong-enterprise-license -n kong-dp --from-file=license=/etc/kong/license.json
secret/kong-enterprise-license created
KongEduLab:~/KGLL-212$ pwd
/home/ubuntu/KGLL-212
KongEduLab:~/KGLL-212$ ll
total 56
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:30 ./
drwxr-xr-x  9 ubuntu ubuntu 4096 Jul  9 09:22 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
-rw-rw-r--  1 ubuntu ubuntu  136 Jul  9 09:29 admin_gui_session_conf
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:20 base/
-rw-rw-r--  1 ubuntu ubuntu  676 Jul  9 09:26 cluster.crt
-rw-------  1 ubuntu ubuntu  306 Jul  9 09:26 cluster.key
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
-rw-rw-r--  1 ubuntu ubuntu  177 Jul  9 09:30 portal_gui_session_conf
KongEduLab:~/KGLL-212$ tree base
base
├── cp-values.yaml
├── cp-values3.yaml
├── deploy-kic.sh
├── dp-values.yaml
├── dp-values3.yaml
├── httpbin-ingress.yaml
├── httpbin.yaml
├── install.sh
├── kic-workspace-a.yaml
├── kic-workspace-b.yaml
├── kic-workspace-c.yaml
├── kic-workspace-d.yaml
├── kind-config-before-i-started-messing.yaml
├── kind-config.yaml
├── misc
│   └── kong_realm_template.json
├── nginx
│   ├── custom_nginx.template
│   ├── docker-compose.yaml
│   ├── nginx_kong.lua
│   └── nginx_kong.lua.1
├── patch.sh
├── reset-lab.sh
└── teardown.sh

2 directories, 22 files
KongEduLab:~/KGLL-212$ helm repo add kong https://charts.konghq.com
"kong" has been added to your repositories
KongEduLab:~/KGLL-212$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "kong" chart repository
Update Complete. ⎈Happy Helming!⎈
KongEduLab:~/KGLL-212$ yq -i '.env.admin_gui_url = env(KONG_ADMIN_GUI_URL)' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.env.admin_api_url = env(KONG_ADMIN_API_URL)' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.env.admin_api_uri = env(KONG_ADMIN_API_URI)' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.env.proxy_url = env(KONG_PROXY_URL)' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.env.portal_api_url = env(KONG_PORTAL_API_URL)' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.env.portal_gui_host = env(KONG_PORTAL_GUI_HOST)' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq '.env' ./base/cp-values.yaml
audit_log: "on"
vitals: on
# vitals_strategy: prometheus
# vitals_statsd_address: statsd-prometheus-statsd-exporter.monitoring.svc.cluster.local:9125
# vitals_tsdb_address: prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
database: "postgres"
role: "control_plane"
cluster_cert: "/etc/secrets/kong-cluster-cert/tls.crt"
cluster_cert_key: "/etc/secrets/kong-cluster-cert/tls.key"
status_listen: 0.0.0.0:8100
portal_gui_protocol: "https"
portal_auth: basic-auth
portal_session_conf:
  valueFrom:
    secretKeyRef:
      name: kong-portal-session-config
      key: portal_session_conf
password:
  valueFrom:
    secretKeyRef:
      name: kong-enterprise-superuser-password
      key: password
admin_gui_url: https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30500
admin_api_uri: ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30501
proxy_url: http://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30000
portal_api_url: https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30444
portal_gui_host: ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30446
admin_api_url: https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30501
portal_gui_ssl_cert: /etc/secrets/kong-manager-tls/tls.crt
portal_gui_ssl_cert_key: /etc/secrets/kong-manager-tls/tls.key
admin_gui_ssl_cert: /etc/secrets/kong-manager-tls/tls.crt
admin_gui_ssl_cert_key: /etc/secrets/kong-manager-tls/tls.key
admin_ssl_cert: /etc/secrets/kong-admin-tls/tls.crt
admin_ssl_cert_key: /etc/secrets/kong-admin-tls/tls.key
KongEduLab:~/KGLL-212$ env|grep KONG_
KONG_ADMIN_API_URI=ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30501
KONG_ADMIN_API_URL=https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30501
KONG_PORTAL_GUI_HOST=ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30446
KONG_PROXY_URI=http://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30000
KONG_PROXY_URL=http://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30000
KONG_ADMIN_GUI_URL=https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30500
KONG_PORTAL_API_URL=https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30444
KONG_COURSE_ID=KGLL-212
KONG_DEMO_PROXY=http://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:31112
KongEduLab:~/KGLL-212$ helm install -f ./base/cp-values.yaml kong kong/kong -n kong \
>     --set manager.ingress.hostname=${FQDN} \
>     --set portal.ingress.hostname=${FQDN} \
>     --set admin.ingress.hostname=${FQDN} \
>     --set portalapi.ingress.hostname=${FQDN}
NAME: kong
LAST DEPLOYED: Sun Jul  9 09:40:48 2023
NAMESPACE: kong
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:

HOST=$(kubectl get svc --namespace kong kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=$(kubectl get svc --namespace kong kong-kong-proxy -o jsonpath='{.spec.ports[0].port}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/
KongEduLab:~/KGLL-212$ ll
total 56
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:30 ./
drwxr-xr-x 10 ubuntu ubuntu 4096 Jul  9 09:38 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
-rw-rw-r--  1 ubuntu ubuntu  136 Jul  9 09:29 admin_gui_session_conf
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:39 base/
-rw-rw-r--  1 ubuntu ubuntu  676 Jul  9 09:26 cluster.crt
-rw-------  1 ubuntu ubuntu  306 Jul  9 09:26 cluster.key
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
-rw-rw-r--  1 ubuntu ubuntu  177 Jul  9 09:30 portal_gui_session_conf
KongEduLab:~/KGLL-212$ watch "kubectl get pods -n kong"
KongEduLab:~/KGLL-212$ helm install -f ./base/dp-values.yaml kong-dp kong/kong -n kong-dp \
> --set proxy.ingress.hostname=${FQDN}
NAME: kong-dp
LAST DEPLOYED: Sun Jul  9 10:14:03 2023
NAMESPACE: kong-dp
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:
HOST=$(kubectl get nodes --namespace kong-dp -o jsonpath='{.items[0].status.addresses[0].address}')
PORT=$(kubectl get svc --namespace kong-dp kong-dp-kong-proxy -o jsonpath='{.spec.ports[0].nodePort}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/
KongEduLab:~/KGLL-212$ watch "kubectl get pods -n kong-dp"
KongEduLab:~/KGLL-212$ kubectl apply -f ./base/httpbin.yaml && kubectl apply -f ./base/httpbin-ingress.yaml 
deployment.apps/httpbin created
service/httpbin created
namespace/httpbin-demo created
service/httpbin-service created
ingress.networking.k8s.io/httpbin-ingress created
KongEduLab:~/KGLL-212$ http get $KONG_PROXY_URL/httpbin
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 394
Content-Type: application/json
Date: Sun, 09 Jul 2023 10:16:18 GMT
Server: gunicorn/19.9.0
Via: kong/2.7.2.0-enterprise-edition
X-Kong-Proxy-Latency: 1
X-Kong-Upstream-Latency: 4

{
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Connection": "keep-alive",
        "Host": "ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30000",
        "User-Agent": "HTTPie/1.0.3",
        "X-Forwarded-Host": "ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io",
        "X-Forwarded-Path": "/httpbin",
        "X-Forwarded-Prefix": "/httpbin"
    }
}

KongEduLab:~/KGLL-212$ http --verify no get https://$FQDN:30443/httpbin
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Access-Control-Allow-Origin: *
Connection: keep-alive
Content-Length: 394
Content-Type: application/json
Date: Sun, 09 Jul 2023 10:19:02 GMT
Server: gunicorn/19.9.0
Via: kong/2.7.2.0-enterprise-edition
X-Kong-Proxy-Latency: 1
X-Kong-Upstream-Latency: 2

{
    "headers": {
        "Accept": "*/*",
        "Accept-Encoding": "gzip, deflate",
        "Connection": "keep-alive",
        "Host": "ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30443",
        "User-Agent": "HTTPie/1.0.3",
        "X-Forwarded-Host": "ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io",
        "X-Forwarded-Path": "/httpbin",
        "X-Forwarded-Prefix": "/httpbin"
    }
}

KongEduLab:~/KGLL-212$ yq -i '.kong-addr = env(KONG_ADMIN_API_URL)' ./deck/deck.yaml 
KongEduLab:~/KGLL-212$ ll
total 56
drwxr-xr-x  8 ubuntu ubuntu 4096 Jul  9 09:30 ./
drwxr-xr-x 10 ubuntu ubuntu 4096 Jul  9 09:38 ../
-rw-------  1 ubuntu ubuntu 1024 Jul  9 09:26 .rnd
-rw-rw-r--  1 ubuntu ubuntu  136 Jul  9 09:29 admin_gui_session_conf
drwxr-xr-x  4 ubuntu ubuntu 4096 Jul  9 09:39 base/
-rw-rw-r--  1 ubuntu ubuntu  676 Jul  9 09:26 cluster.crt
-rw-------  1 ubuntu ubuntu  306 Jul  9 09:26 cluster.key
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 commands/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 10:25 deck/
drwxr-xr-x  2 ubuntu ubuntu 4096 Jul  9 09:02 docker-containers/
drwxr-xr-x 14 ubuntu ubuntu 4096 Jul  9 09:02 exercises/
drwxr-xr-x  3 ubuntu ubuntu 4096 Jul  9 09:02 files/
-rw-r--r--  1 ubuntu ubuntu 1691 Jul  9 09:02 install_kic.sh
-rw-rw-r--  1 ubuntu ubuntu  177 Jul  9 09:30 portal_gui_session_conf
KongEduLab:~/KGLL-212$ tree deck
deck
└── deck.yaml

0 directories, 1 file
KongEduLab:~/KGLL-212$ cat deck/deck.yaml 
kong-addr: https://ndojgmtyoezgtamla-63e39400016d14f72e8da552.labs.strigo.io:30501
headers:
  - "key1:value1"
no-color: false
verbose: 0
KongEduLab:~/KGLL-212$ deck ping --config deck/deck.yaml
Successfully connected to Kong!
Kong version:  2.7.2.0-enterprise-edition
KongEduLab:~/KGLL-212$ deck dump --config deck/deck.yaml --output-file deck/preupgrade.yaml
KongEduLab:~/KGLL-212$ l
admin_gui_session_conf  base/  cluster.crt  cluster.key  commands/  deck/  docker-containers/  exercises/  files/  install_kic.sh  portal_gui_session_conf
KongEduLab:~/KGLL-212$ tree ./deck
./deck
├── deck.yaml
└── preupgrade.yaml

0 directories, 2 files
KongEduLab:~/KGLL-212$ cat deck/preupgrade.yaml 
_format_version: "1.1"
services:
- connect_timeout: 60000
  enabled: true
  host: httpbin-service.httpbin-demo.80.svc
  name: httpbin-demo.httpbin-service.pnum-80
  path: /headers
  port: 80
  protocol: http
  read_timeout: 60000
  retries: 5
  routes:
  - https_redirect_status_code: 426
    name: httpbin-demo.httpbin-ingress.00
    path_handling: v0
    paths:
    - /httpbin$
    - /httpbin/
    preserve_host: true
    protocols:
    - http
    - https
    regex_priority: 200
    request_buffering: true
    response_buffering: true
    strip_path: true
    tags:
    - managed-by-ingress-controller
  tags:
  - managed-by-ingress-controller
  write_timeout: 60000
upstreams:
- algorithm: round-robin
  hash_fallback: none
  hash_on: none
  hash_on_cookie_path: /
  healthchecks:
    active:
      concurrency: 10
      healthy:
        http_statuses:
        - 200
        - 302
        interval: 0
        successes: 0
      http_path: /
      https_verify_certificate: true
      timeout: 1
      type: http
      unhealthy:
        http_failures: 0
        http_statuses:
        - 429
        - 404
        - 500
        - 501
        - 502
        - 503
        - 504
        - 505
        interval: 0
        tcp_failures: 0
        timeouts: 0
    passive:
      healthy:
        http_statuses:
        - 200
        - 201
        - 202
        - 203
        - 204
        - 205
        - 206
        - 207
        - 208
        - 226
        - 300
        - 301
        - 302
        - 303
        - 304
        - 305
        - 306
        - 307
        - 308
        successes: 0
      type: http
      unhealthy:
        http_failures: 0
        http_statuses:
        - 429
        - 500
        - 503
        tcp_failures: 0
        timeouts: 0
    threshold: 0
  name: httpbin-service.httpbin-demo.80.svc
  slots: 10000
  tags:
  - managed-by-ingress-controller
  targets:
  - tags:
    - managed-by-ingress-controller
    target: httpbin.default.svc.cluster.local:80
    weight: 100
KongEduLab:~/KGLL-212$ yq -i '.ingressController.image.tag = "2.5"' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.image.tag = "2.8.1.1-alpine"' ./base/dp-values.yaml
KongEduLab:~/KGLL-212$ yq -i '.image.tag = "2.8.1.1-alpine"' ./base/cp-values.yaml
KongEduLab:~/KGLL-212$ yq '.ingressController.image.tag' ./base/cp-values.yaml
2.5
KongEduLab:~/KGLL-212$ yq '.image.tag' ./base/cp-values.yaml
2.8.1.1-alpine
KongEduLab:~/KGLL-212$ yq '.image.tag' ./base/dp-values.yaml
2.8.1.1-alpine
KongEduLab:~/KGLL-212$ helm upgrade -f ./base/dp-values.yaml kong-dp kong/kong -n kong-dp \
> --set proxy.ingress.hostname=$FQDN --wait
Release "kong-dp" has been upgraded. Happy Helming!
NAME: kong-dp
LAST DEPLOYED: Sun Jul  9 10:27:39 2023
NAMESPACE: kong-dp
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:
HOST=$(kubectl get nodes --namespace kong-dp -o jsonpath='{.items[0].status.addresses[0].address}')
PORT=$(kubectl get svc --namespace kong-dp kong-dp-kong-proxy -o jsonpath='{.spec.ports[0].nodePort}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/
KongEduLab:~/KGLL-212$ helm upgrade -f ./base/cp-values.yaml kong kong/kong -n kong \
> --set manager.ingress.hostname=$FQDN \
> --set portal.ingress.hostname=$FQDN \
> --set admin.ingress.hostname=$FQDN \
> --set portalapi.ingress.hostname=$FQDN --wait
Release "kong" has been upgraded. Happy Helming!
NAME: kong
LAST DEPLOYED: Sun Jul  9 10:28:49 2023
NAMESPACE: kong
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:

HOST=$(kubectl get svc --namespace kong kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=$(kubectl get svc --namespace kong kong-kong-proxy -o jsonpath='{.spec.ports[0].port}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://docs.konghq.com/kubernetes-ingress-controller/latest/guides/getting-started/
KongEduLab:~/KGLL-212$ watch "kubectl get pods -A"
KongEduLab:~/KGLL-212$ http get $KONG_ADMIN_API_URL | jq .version
"2.8.1.1-enterprise-edition"
KongEduLab:~/KGLL-212$ deck sync --config deck/deck.yaml -s deck/preupgrade.yaml
Summary:
  Created: 0
  Updated: 0
  Deleted: 0

```

