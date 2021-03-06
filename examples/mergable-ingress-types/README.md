# Mergeable Ingress Types Support

You can configure NGINX to have support for using multiple Ingress Resources which apply to a common host. This allows
for easier management when using a large amount of paths.

## Syntax and Rules

A Master is declared using `nginx.org/mergible-ingress-type: master`. A Master will process all configurations at the
host level, which includes the TLS configuration, and any annotations which will be applied for the complete host. There
can only be one ingress resource on a unique host that contains the master value. Paths cannot be part of the
ingress resource.

A Minion is declared using `nginx.org/mergible-ingress-type: minion`. A Minion will be used to append different
locations to an ingress resource with the Master value. The annotations of minions are replaced with the annotations of
their master. TLS configurations are not allowed. There can be multiple minions which must have the same host as the master.

Note: Ingress Resources with more than one host cannot be used.

## Example

In this example we deploy the NGINX Ingress controller, a simple web application and then configure
load balancing for that application using Ingress resources with the `nginx.org/mergible-ingress-type` annotations.

## Running the Example

## 1. Deploy the Ingress Controller

1. Follow the installation [instructions](../../docs/installation.md) to deploy the Ingress controller.

2. Save the public IP address of the Ingress controller into a shell variable:
    ```
    $ IC_IP=XXX.YYY.ZZZ.III
    ```

## 2. Deploy the Cafe Application

Create the coffee and the tea deployments and services:
```
$ kubectl create -f cafe.yaml
```

## 3. Configure Load Balancing

1. Create a secret with an SSL certificate and a key:
    ```
    $ kubectl create -f cafe-secret.yaml
    ```

2. Create the Master Ingress resource:
    ```
    $ kubectl create -f cafe-master.yaml
    ```
    
3. Create the Minion Ingress resource for the Coffee Service:
    ```
    $ kubectl create -f coffee-minion.yaml
    ```

4. Create the Minion Ingress resource for the Tea Service:
   ```
    $ kubectl create -f tea-minion.yaml
    ```

## 4. Test the Application

1. To access the application, curl the coffee and the tea services. We'll use ```curl```'s --insecure option to turn off certificate verification of our self-signed
certificate and the --resolve option to set the Host header of a request with ```cafe.example.com```
    
    To get coffee:
    ```
    $ curl --resolve cafe.example.com:443:$IC_IP https://cafe.example.com/coffee --insecure
    Server address: 10.12.0.18:80
    Server name: coffee-7586895968-r26zn
    ...
    ```
    If you prefer tea:
    ```
    $ curl --resolve cafe.example.com:443:$IC_IP https://cafe.example.com/tea --insecure
    Server address: 10.12.0.19:80
    Server name: tea-7cd44fcb4d-xfw2x
    ...
    ```

    **Note**: If you're using a NodePort service to expose the Ingress controller, replace port 443 in the commands above with the node port that corresponds to port 443.
    
## 5. Examine the Configuration

1. Access the NGINX Pod.
```
$ kubectl get pods -n nginx-ingress
NAME                             READY     STATUS    RESTARTS   AGE
nginx-ingress-66bc44674b-hrcx8   1/1       Running   0          4m
```

2. Examine the NGINX Configuration.
```
$ kubectl exec -it nginx-ingress-66bc44674b-hrcx8 -n nginx-ingress cat /etc/nginx/conf.d/default-cafe-ingress-master.conf

upstream default-cafe-ingress-coffee-minion-cafe.example.com-coffee-svc {
	server 172.17.0.5:80;
	server 172.17.0.6:80;
}
upstream default-cafe-ingress-tea-minion-cafe.example.com-tea-svc {
	server 172.17.0.7:80;
	server 172.17.0.8:80;
	server 172.17.0.9:80;	
}
 # *Master*, configured in Ingress Resource: default-cafe-ingress-master
server {
	listen 80;
	listen 443 ssl;
	ssl_certificate /etc/nginx/secrets/default-cafe-secret;
	ssl_certificate_key /etc/nginx/secrets/default-cafe-secret;
	server_tokens on;
	server_name cafe.example.com;
	if ($scheme = http) {
		return 301 https://$host:443$request_uri;
	}
	 # *Minion*, configured in Ingress Resource: default-cafe-ingress-coffee-minion
	location /coffee {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		proxy_pass http://default-cafe-ingress-coffee-minion-cafe.example.com-coffee-svc;	
	}
	 # *Minion*, configured in Ingress Resource: default-cafe-ingress-tea-minion
	location /tea {
		proxy_http_version 1.1;
		proxy_connect_timeout 60s;
		proxy_read_timeout 60s;
		client_max_body_size 1m;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Host $host;
		proxy_set_header X-Forwarded-Port $server_port;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering on;
		proxy_pass http://default-cafe-ingress-tea-minion-cafe.example.com-tea-svc;
	}
}
```
