# Customizing Grafana login page

This document describe how to customize Grafana Login page using **[HAPROXY](https://github.com/haproxy/haproxy)** without changing Grafana code

**Login Page**
![Login page](grafana_login.png)

### Prerequisites

I did the tests on a version of Grafana 6.5.3, which is a older version that does not provide the customization of images from the grafana configuration file (grafana.ini)

```
Grafana
HAPROXY
Docker
lite-server 
Ubuntu
```
## Install

### Installing and running Grafana

I used a docker solution to run Grafana  
At First we need to make a copy of grafana.ini file and we need to change a couple of parameters as they are described in this [link](https://grafana.com/tutorials/run-grafana-behind-a-proxy/)

```
[server]
domain = localhost
root_url = %(protocol)s://%(domain)s:%(http_port)s/grafana/
serve_from_sub_path = true
```

And than Run grafana docker image using our custom file grafana.ini

```
docker run --rm  -p 3000:3000 --mount type=bind,source=$PWD/grafana/defaults.ini,target=/etc/grafana/grafana.ini  --name=grafana -e "GF_SERVER_ROOT_URL=http://localhost:3000/grafana" -e "GF_SERVER_SERVE_FROM_SUB_PATH=true" --name grafana grafana/grafana:6.5.3
```

check if Grafana is running correctly on port 3000 http://localhost:3000/

### Installing and running lite sever

copy **img** directory from **/grafana/public/img**
for this setup i copied img folder to **/var/www/html/grafana/**

run lite-server Docker image on port 5000 from this directory
```
cd /var/www/html/grafana/
docker run --rm --name lite-server -p 5000:3000 -v $(pwd):/src gecko8/lite-server
```

### Installing and running HAPROXY

Install HAPROXY on your Ubuntu or you can use a dockerized solution for HAPROXY
```
sudo apt-get update
sudo apt-get install haproxy
```

Edit haproxy.cfg file and define and setup rules for serving static images

```
vi /etc/haproxy/haproxy.cfg 
```
**frontend section**

```
frontend grafana-localhost
  	bind *:80
	log global

	#################################################
	############1-ACL ###############################
	#################################################
	# LOCALHOST
	acl is_local_host src 192.168.40.139
	acl is_local_host src localhost
	acl is_local_host src 127.0.0.1
        acl grafana_static_img  path_beg         /grafana/public/img/
	acl grafana_path path_beg 	 /grafana/
	acl grafana_path path	 	 /grafana
        acl host_grafana    hdr_beg(host) -i 192.168.40.139
	acl host_grafana    hdr_beg(host) -i localhost:3000

	#################################################
	##########2-LOG FORMATS      ####################
	#################################################
	log-format '%ci:%cp %HM %HU %b/%s'

	#################################################
	##########3-SWITCH between BACKENDS #############
	#################################################
	use_backend static_grafana if host_grafana grafana_static_img
	use_backend grafana_backend if is_local_host grafana_path 
```

**I defined three backends for grafana**
**1. backend grafana_backend_localhost** for localhost grafana server (it's my grafana docker running on port 3000)
  grafana_backend_localhost not really needed( used for test changes)
  
```
  backend grafana_backend_localhost
        server grafana localhost:3000
	    log global
```  
**2. backend grafana_backend**  it's our proxied Grafana    

```
backend grafana_backend
  	http-request set-path %[path,regsub(^/grafana/?,/)]
    server grafana localhost:3000
	log global
``` 

**3. backend static_grafana**  for our copy of /public/img Grafana directory  and our custom images
  here i set the rules for serving images 
```
backend static_grafana
	acl grafana_icon_svg path_end -i /grafana/public/img/grafana_icon.svg
	acl grafana_typelogo_svg path_end -i /grafana/public/img/grafana_typelogo.svg
	acl heatmap_bg_test_svg path_end -i /grafana/public/img/heatmap_bg_test.svg
	http-request set-path %[path,regsub(^/grafana/public/img/grafana_icon.svg?,'/img/biznet.svg')] if grafana_icon_svg
	http-request set-path %[path,regsub(^/grafana/public/img/grafana_typelogo.svg?,'/img/custom_text.svg')] if grafana_typelogo_svg
	http-request set-path %[path,regsub(^/grafana/public/img/heatmap_bg_test.svg?,'/img/sfondoblu2.jpg')] if heatmap_bg_test_svg
	http-request set-path %[path,regsub(^/grafana/public?,'')]

	server grafana_static localhost:5000
```

A full copy of my **[haproxy.cfg](haproxy.cfg)**

Restart Haproxy for applying changes
```
sudo service haproxy restart
```

For Haproxy logs 
```
sudo tail -f /var/log/haproxy.log
```

### Results
**login page**
![Presentation](grafana.gif)

**lite server log**

![log](lite-server-log.png)

## Authors

* **Ahmad Nazha** 

