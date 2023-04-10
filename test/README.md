# kubernets-chart-httpd-ihs

## Build the image httpd-ihs

### Get the httpd image & get the httpd file from the image
```
docker pull httpd:2.4
docker run -d --name httpd httpd:2.4
docker cp httpd:/usr/local/apache2/conf/httpd.conf .
```

### Add the config to httpd file
```
tee httpd.conf<<EOF
# ******************************************************************
# * IHS apache SSL config for client.dijon.fr for docker image
# ******************************************************************
LoadModule ssl_module modules/mod_ssl.so
LoadModule rewrite_module modules/mod_rewrite.so

<IfModule mod_ssl.c>
  #/usr/local/apache2
  #bin  build  cgi-bin  conf  error  htdocs  icons  include  logs modules
  Listen 443

  <VirtualHost *:443>
    ServerAdmin admin@myexample.com
    DocumentRoot /usr/local/apache2/htdocs
    ServerName ihs.dijon.fr
    ServerAlias ihs-dijon.fr

    Alias /ihs "/usr/local/apache2/htdocs"

    <Directory /usr/local/apache2/htdocs>
      Options +FollowSymlinks
       AllowOverride All
       Require all granted
       <IfModule mod_dav.c>
         Dav off
       </IfModule>
       SetEnv HOME /usr/local/apache2/htdocs
       SetEnv HTTP_HOME /usr/local/apache2/htdocs

       RewriteEngine   on
       SSLVerifyClient optional_no_ca
       LogLevel warn rewrite:trace8
       RewriteCond    "%{SSL:SSL_CLIENT_S_DN_CN}" "!^(.*client.dijon.fr)$"
       RewriteRule    "^.*" "-" [R=403]

       #RewriteRule    "^.*"  "https://%{HTTP_HOST}/index-notok.html" [R=301,L]
    </Directory>

    SSLCertificateFile /etc/ssl/certs/ihs.dijon.fr.crt
    SSLCertificateKeyFile /etc/ssl/private/ihs.dijon.fr.key
  </VirtualHost>
</IfModule>
EOF
```

### Dockerfile
```
tee Dockerfile<<EOF
FROM httpd:2.4

#COPY ./public-html/ /usr/local/apache2/htdocs/
#/usr/local/apache2/conf

RUN mkdir -p /etc/ssl/certs /etc/ssl/private
#RUN mkdir /usr/local/apache2/ihs
#RUN chown -R www-data.www-data /usr/local/apache2/ihs

COPY  index-ok.html index-notok.html /usr/local/apache2/htdocs/
RUN chown www-data:www-data /usr/local/apache2/htdocs/index*.html

COPY ihs.dijon.fr.crt  /etc/ssl/certs/
COPY ihs.dijon.fr.key /etc/ssl/private
COPY httpd.conf /usr/local/apache2/conf

EXPOSE 80
EXPOSE 443
EOF
```

### Build & tests
```
docker build . -t httpd-ihs:0.1   ;docker stop httpd-ihs ; docker rm httpd-ihs ;docker run -d --name httpd-ihs -p 9090:80 -p 9091:443  httpd-ihs:0.1; docker ps

docker tag httpd-ihs:0.1   davbou/httpd-ihs:0.1

$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username:
Password:

docker push davbou/httpd-ihs:0.1
```

# Tests CN authentication should be .*client.dijon.fr or redirect Error 403

```
$ curl -v -k -L --cert /root/kubernetes/apachessl/donotcommit/client.dijon.fr/client.dijon.fr.crt --key /root/kubernetes/apachessl/donotcommit/client.dijon.fr/client.dijon.fr.key    https://localhost:9091

[Wed Oct 06 15:02:37.893480 2021] [rewrite:trace4] [pid 11:tid 140291104044800] mod_rewrite.c(480): [client 172.17.0.1:45786] 172.17.0.1 - - [localhost/sid#7f9812742e68][rid#7f98106680a0/initial] [perdir /usr/local/apache2/htdocs/] RewriteCond: input='client.dijon.fr' pattern='!^(.*client.dijon.fr)$' => not-matched


$ curl -v -k -L --cert ./donotcommit/ssl/beats/beat.crt --key ./donotcommit/ssl/beats/beat.key --cacert ./donotcommit/ssl/ca/ca.crt    https://localhost:9091

[Wed Oct 06 15:02:43.803151 2021] [rewrite:trace4] [pid 10:tid 140291104044800] mod_rewrite.c(480): [client 172.17.0.1:45790] 172.17.0.1 - - [localhost/sid#7f9812742e68][rid#7f98106680a0/initial] [perdir /usr/local/apache2/htdocs/] RewriteCond: input='beats.myelk.com' pattern='!^(.*client.dijon.fr)$' => matched
```

## Manualy install
```
$ cat deployment-httpd-ihs.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-ihs-dijon-fr
  labels:
    app: httpd-ihs
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: httpd-ihs
  template:
    metadata:
      labels:
        app: httpd-ihs
    spec:
      containers:
      - name: httpd-ihs-dijon-fr
        image: davbou/httpd-ihs:0.1
        ports:
        - containerPort: 80
        - containerPort: 443
      imagePullSecrets:
      - name: regcred

$ cat service-nodeport-httpd-ihs.yml
apiVersion: v1
kind: Service
metadata:
  name: httpd-ihs-dijon-fr
spec:
  selector:
    app: httpd-ihs
  type: NodePort
  ports:
  - port: 443
    targetPort: 443

kubectl -n dev apply -f deployment-httpd-ihs.yml
kubectl -n dev apply -f service-nodeport-httpd-ihs.yml
```

## helm
```
helm -n dev install httpd-ihs-dijon-fr . -f values-dijon-fr.yaml

helm -n dev list
NAME              	NAMESPACE	REVISION	UPDATED                                 	STATUS  	CHART          	APP VERSION
httpd-ihs-dijon-fr	dev     	1       	2021-10-07 12:07:20.188299815 +0200 CEST	deployed	httpd-ihs-0.1.0	1.16.0
```
