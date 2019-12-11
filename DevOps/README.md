# Docker+Jenkins + Gitlab

>CentOS7.*

## [install docker-ce](../docker/README.md)

## install gitlab
```
docker pull gitlab/gitlab-ce

docker run --detach \
--hostname gitlab.xxxx.com \
--publish 8443:443 --publish 880:80 --publish 822:22 \
--name gitlab \
--restart always \
--volume ~/gitlab/config:/etc/gitlab \
--volume ~/gitlab/logs:/var/log/gitlab \
--volume ~/gitlab/data:/var/opt/gitlab \
gitlab/gitlab-ce

vim  /root/gitlab/config/gitlab.rb

external_url 'https://gitlab.xxxx.com'  #配置HTTPS地址
nginx['redirect_http_to_https'] = true  #开启HTTP强制定向到HTTPS功能
[email]
docker exec -t -i gitlab vim /etc/gitlab/gitlab.rb
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.163.com"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "xxxx@163.com"
gitlab_rails['smtp_password'] = "xxxxpassword"
gitlab_rails['smtp_domain'] = "163.com"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = false
gitlab_rails['smtp_openssl_verify_mode'] = "peer"
gitlab_rails['gitlab_email_from'] = "xxxx@163.com"
user["git_user_email"] = "xxxx@163.com"

[nginx]
server{
    listen          443 ssl;
    server_name     gitlab.xxxx.com;
    index           index.html index.htm;
    ssl on;
    ssl_certificate         /etc/nginx/ssl/gitlab.crt;
    ssl_certificate_key    /etc/nginx/ssl/gitlab.key;
    root            /var/www;
    location / {
    client_max_body_size 0;
    gzip off;

    proxy_read_timeout      300;
    proxy_connect_timeout   300;
    proxy_redirect          off;
    proxy_http_version 1.1;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto   $scheme;
    proxy_pass https://127.0.0.1:8443;
    }
    #access_log_bypass_if ($status = 403);
    access_log /var/log/nginx/$host.log log;
    error_log /var/log/nginx/$host.log warn;
}

docker restart gitlab
```
## Jenkins
```
docker run -d \
-p 8080:8080 \
-p 50000:50000 \
--name jenkins \
--link gitlab:gitlab.xxxx.com \
-u root \
-v ~/jenkins:/var/jenkins_home  \
jenkinsci/jenkins:2

upstream jenkins {
  keepalive 32; # keepalive connections
  server 127.0.0.1:8080; # jenkins ip and port
}
server{
    listen          80;
    server_name    jenkins.xxxx.com;
    index           index.html index.htm;
	 #this is the jenkins web root directory (mentioned in the /etc/default/jenkins file)
    root           /var/jenkins/war/;
    ignore_invalid_headers off; #pass through headers from Jenkins which are considered invalid by Nginx server.	
	
	location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
	#rewrite all static files into requests to the root
	#E.g /static/12345678/css/something.css will become /css/something.css
	rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
	}
    location /userContent {
	#have nginx handle all the static requests to the userContent folder files
	#note : This is the $JENKINS_HOME dir
	root /var/jenkins/;
	if (!-f $request_filename){
	  #this file does not exist, might be a directory or a /**view** url
	  rewrite (.*) /$1 last;
	  break;
	}
	sendfile on;
	}
    location / {

 sendfile off;
      proxy_pass         http://jenkins;
      proxy_redirect     default;
      proxy_http_version 1.1;

      proxy_set_header   Host              $host;
      proxy_set_header   X-Real-IP         $remote_addr;
      proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
proxy_max_temp_file_size 0;

      #this is the maximum upload size
      client_max_body_size       10m;
      client_body_buffer_size    128k;

      proxy_connect_timeout      90;
      proxy_send_timeout         90;
      proxy_read_timeout         90;
      proxy_buffering            off;
      proxy_request_buffering    off; # Required for HTTP CLI commands in Jenkins > 2.54
      proxy_set_header Connection ""; # Clear for keepalive
    }
    #access_log_bypass_if ($status = 403);
 
}
```
### [参考资料](https://blog.csdn.net/guyan0319/article/details/89290416)



