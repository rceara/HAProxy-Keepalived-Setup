# HAProxySetup
# Setup HAProxy as load balancers for Model Driven Telemetry with Cisco gear #

## Before we start our journey installing and configuring HAProxy I would like to share some thoughts about the power of HAProxy as an OpenSource Load Balancer. ##

Do we need a Layer 4 or 7 Proxy or Load Balancer to integrate with your gRPC collectors to monitor your Cisco WLCs?
The answer is: Yes! To be able to grow your deployment at scale and manage multiple collectors its preferable to use a load balancer in the frontend and collectors in the backend.

It's there a Load Balancer I can use to play around with the solution to integrated them with my Cisco gear?
The answers is: Yes. HAProxy is a pretty good open source Load Balancer.

A Proxy or Load Balancer like HAProxy helps with the redistribution of the traffic that is coming to your backend servers via HTTP and TCP. There are multiple reasons why we should give preferences to a  Network/Application Load Balancer:

1. HA Proxy is opensource (there is also an Enterprise version with 24/7 SLA support) that offer a lot of flexibility and customization. You can read in the following link the documentation: https://cbonte.github.io/haproxy-dconv/1.7/configuration.html
  
2. HAProxy offers Security and Reliability. Reliability most be the strongest attributes of HAProxy because it’s a very mature and robust Load Balancer that can handle thousands of sessions and can help in the encryption of yours HTTP and TCP sessions with TLS/mTLS tunnels between the clients and the servers.

3. Flexibility of deployment. There are multiple ways and scenarios of how you can deploy the solution. HAProxy can run on BareMetal, Containers and Virtual Machines. It can act as a reverse proxy, passthrough and load balancer on active/standby, active/active and multi-layer load balancers to guarantee the failover of the solution.

4. HA Proxy also offer data plane API integration and has a basic web-interface for stats (the Enterprise version its very powerful) that can be customized and also integrates with multiple opensource tools such as: Nagios, Cacti, Zabbix, Telegraf, Grafana, Prometheus, etc.

5. Last but not least, HAProxy hast a pretty good support from the community.
Links to access the community: http://discourse.haproxy.org/  & https://www.mail-archive.com/haproxy@formilux.org/

## Now that we have done an introduction to HAProxy, I would like to get into the details of how to install and configure HAProxy. ##

Lets go through the configuration and integration process of HAProxy with your gRPC collectors to monitor your Cisco Infrastructure:

Install HAProxy:
```
root@collector3:/etc/haproxy# apt install haproxy
```
Verify HAProxy config to see if there is any errors:
```
root@collector3:/etc/haproxy# haproxy -f /etc/haproxy/haproxy.cfg -c
```
Start and verify the status of HAProxy:
```
root@collector3:/etc/haproxy# systemctl start haproxy
```
```
root@collector3:/etc/haproxy# systemctl status haproxy
```

Verify HAProxy Logs:
```
root@collector3:/etc/haproxy# tail -F /var/log/haproxy.log
```
All HAProxy instances in the cluster should have the same configuration for peers settings. After you've restarted the 
HAProxy service on each server, you can make a request to the website so that an entry is added to the table and then use socket commands to check that the data is replicated. The following command would show you the entries stored in the grpc_servers table on the primary: 
```
root@collector3:/etc/haproxy# echo "show table grpc_servers" | sudo socat stdio /run/haproxy/admin.sock
# table: grpc_servers, type: ip, size:1048576, used:1
0x555647bbfe30: key=10.93.178.70 use=0 exp=1706183 server_id=1 server_key=s1
```
## Here is a snapshot Configuration of HA Proxy1: ##
```
root@collector3:~# more /etc/haproxy/haproxy.cfg
```
```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-
GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	tcp
	option  tcplog
        option	dontlognull
        timeout connect 30s
        timeout client  60m
        timeout server  60m
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http

listen  stats
        bind		10.93.178.155:1936
        mode            http
        log             global
        stats enable
        stats hide-version
        stats refresh 30s
        stats show-node
        stats auth netadmin:C1sco12345
        stats uri  /haproxy?stats
	stats show-desc Load-Balancer1
	stats show-legends
	stats admin if TRUE	# Thestatspage is great for seeing the status of our backend servers, but we can alsocontrolthemthrough this page. When we add astats admindirective, the page displays a checkbox next to eachserver and a dropdown men
u that lets you perform an action on it

peers mycluster
  peer collector3.mydomainname.com 10.93.178.155:10000
  peer loadbalancer2 10.93.178.173:10000

frontend grpc_service
   mode tcp
   log global
   option logasap
   option tcplog
   bind *:57499
   default_backend grpc_servers

backend grpc_servers
   mode tcp
   #balance source
   balance leastconn
   option log-health-checks
   option ssl-hello-chk
   server s1 10.93.178.157:57499 check
   server s2 10.93.178.175:57499 check
   server s3 10.93.178.176:57499 check
   stick-table type ip size 1m size 1m expire 30m peers mycluster
   stick on src
   #stick match src
```
## Here is a snapshot Configuration of HA Proxy2: ##
```
root@collector3:~# more /etc/haproxy/haproxy.cfg
```
```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

	# Default SSL material locations
	ca-base /etc/ssl/certs
	crt-base /etc/ssl/private

	# See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-
GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
	log	global
	mode	tcp
	option  tcplog
        option	dontlognull
        timeout connect 30s
        timeout client  60m
        timeout server  60m
	errorfile 400 /etc/haproxy/errors/400.http
	errorfile 403 /etc/haproxy/errors/403.http
	errorfile 408 /etc/haproxy/errors/408.http
	errorfile 500 /etc/haproxy/errors/500.http
	errorfile 502 /etc/haproxy/errors/502.http
	errorfile 503 /etc/haproxy/errors/503.http
	errorfile 504 /etc/haproxy/errors/504.http


listen  stats
        bind		10.93.178.173:1936
        mode            http
        log             global
        stats enable
        stats hide-version
        stats refresh 30s
        stats show-node
        stats auth netadmin:C1sco12345
        stats uri  /haproxy?stats
        stats show-desc Load-Balancer1
	stats show-legends
	stats admin if TRUE	# Thestatspage is great for seeing the status of our backend servers, but we can alsocontrolthemthrough this page. When we add astats admin directive, the page displays a checkbox next to eachserver and a dropdown me
nu that lets you perform an action on it

peers mycluster
  peer loadbalancer1 10.93.178.155:10000
  peer collector3.mydomainname.com 10.93.178.173:10000

frontend grpc_service
   mode tcp
   log global
   option logasap
   option tcplog
   bind *:57499
   default_backend grpc_servers

backend grpc_servers
   mode tcp
   #balance source
   balance leastconn
   option log-health-checks
   option ssl-hello-chk
   server s1 10.93.178.157:57499 check
   server s2 10.93.178.175:57499 check
   server s3 10.93.178.176:57499 check
   stick-table type ip size 1m size 1m expire 30m peers mycluster
   stick on src
   #stick match src
```




