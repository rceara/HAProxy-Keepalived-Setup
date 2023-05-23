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

# KeepAlived Setup #
# Setup KeepAlived for the load balancers to send all the traffic from the Cisco gear to a floating of virtual IP #

## Installation and Configuration Keepalived services to work with HAProxy. Make sure you follow this process on Server1 and Server2 of HAProxy. ##

Install keepalived:
```
root@collector3:~# apt install keepalived
```

In order for the Keepalived service to forward network packets properly to the real servers, each router node must have IP forwarding turned on in the kernel:
```
root@collector3:~# vi /etc/sysctl.conf
```
Add the following lines: 
```
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind=1
```

Then run the CLI command: sysctl -p
```
root@collector3:~# sysctl -p
net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
```

## Configure keepalive on HAProxy 1: ##
```
root@collector3:~# more /etc/keepalived/keepalived.conf
# Define the script used to check if haproxy is still working
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2  # check every 2 seconds
    weight -2
    fall 2      # check twice before setting down
    rise 1      # check once before setting up
}

vrrp_instance haproxy1 {
    state MASTER
    interface ens160
    virtual_router_id 101 # Needs to be the same on both Server
    priority 101	  # Higher Priority then master, lower priority then backup
    advert_int 1	#  VRRP advertisment interval
    authentication {
        auth_type PASS	# Use a password to negotiate the vrrp session
        auth_pass C1sco12345
    }
    virtual_ipaddress{
        10.93.178.174	# Define the VIP address to be used
    }

    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }

}
vrrp_instance haproxy2 {
    state BACKUP
    interface ens160
    virtual_router_id 102 # Needs to be the same on both Server
    priority 100          # Higher Priority then master, lower priority then backup
    advert_int 1        #  VRRP advertisment interval
    authentication {
        auth_type PASS  # Use a password to negotiate the vrrp session
        auth_pass C1sco12345
    }
    virtual_ipaddress{
        10.93.178.177   # Define the VIP address to be used
    }
    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }
}
```
## Configure keepalive on HAProxy 2: ##
```
root@collector3:~# more /etc/keepalived/keepalived.conf
# Define the script used to check if haproxy is still working
vrrp_script chk_haproxy {
    script "/usr/bin/killall -0 haproxy"
    interval 2	# check every 2 seconds
    weight -2
    fall 2	# check twice before setting down
    rise 1	# check once before setting up
}

vrrp_instance haproxy1 {
    state BACKUP
    interface ens160
    virtual_router_id 101 # Needs to be the same on both Server
    priority 100	 # Higher Priority then master, lower priority then backup
    advert_int 1	#  VRRP advertisment interval
    authentication {
        auth_type PASS	# Use a password to negotiate the vrrp session
        auth_pass C1sco12345
    }
    virtual_ipaddress{
        10.93.178.174	# Define the VIP address to be used
    }

    }

    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }
}
vrrp_instance haproxy2 {
    state MASTER
    interface ens160
    virtual_router_id 102 # Needs to be the same on both Server
    priority 101         # Higher Priority then master, lower priority then backup
    advert_int 1        #  VRRP advertisment interval
    authentication {
        auth_type PASS  # Use a password to negotiate the vrrp session
        auth_pass C1sco12345
    }
    virtual_ipaddress{
        10.93.178.177   # Define the VIP address to be used
    }
    # Use the Defined Script to Check whether to initiate a fail over
    track_script {
        chk_haproxy
    }
}
```
## Start Keepalive service: ##
```
root@collector3:~# systemctl start keepalived
```
## Verify Keepalive service: ##
```
root@collector3:~#  systemctl status keepalived
```

## Verify floating/virtual IP on the interface. I marked it as ** so you can look for the output when running the CLI command ##
```
root@collector3:~# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:7b:5c:d5 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 10.93.178.155/26 brd 10.93.178.191 scope global ens160
       valid_lft forever preferred_lft forever
   ** inet 10.93.178.174/32 scope global ens160 **
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe7b:5cd5/64 scope link 
       valid_lft forever preferred_lft forever
```


 
