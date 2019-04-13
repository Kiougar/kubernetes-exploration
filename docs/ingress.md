## NGINX Ingress Controller
_[source](https://kubernetes.github.io/ingress-nginx/deploy/)_

### Deploy Ingress NGINX
Add this to the Deployment's Pod spec to **make sure the ingress 
controller pod is deployed in the `master` node**:

```yaml
tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
nodeSelector:
  node-role.kubernetes.io/master: ""
```

### Create Ingress Service

Create a `NodePort` service for the ingress controller:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  externalTrafficPolicy: Local
  ports:
    - name: http
      port: 80
      targetPort: http
      # fixed port so that master can proxy to it
      nodePort: 30080
    - name: https
      port: 443
      targetPort: https
      # fixed port so that master can proxy to it
      nodePort: 30443
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

?> Notice the fixed ports `30080` and `30443` that will be used by the master proxy

### Configure Ingress NGINX

Apply nginx-configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
data:
  # make sure that headers are forwarded from proxy
  use-forwarded-headers: "true"
  # use the master's IP here (i.e the master proxy)
  proxy-real-ip-cidr: "1.2.3.4/32"
```

?> Make sure that you use the correct IP in the `proxy-real-ip-cidr` setting
   otherwise the headers will not be forwarded and **the real client IP will
   never reach the application layer**.

## NGINX Master Proxy

Setup NGINX proxy on master node

### Install NGINX
_the following instructions assume root access_

```bash
apt install nginx
```

### Configure proxy

Replace the contents of the file `/etc/nginx/sites-available/default` with:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;

    include /etc/nginx/snippets/proxy.conf;
}
```

Create the file `/etc/nginx/snippets/proxy.conf` with contents:

```
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_pass http://127.0.0.1:30080;
}
```

?> Notice that it proxies everything to the [Ingress Service](#create-ingress-service)
   while doing TLS termination
   <br>
   _listens to both `80` and `443` and always redirects to `http://127.0.0.1:30080`_

### Secure proxy (optional)

Use [certbot](https://certbot.eff.org/lets-encrypt/ubuntubionic-nginx) to secure the master proxy
