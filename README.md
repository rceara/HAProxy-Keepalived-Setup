# HAProxySetup
# Setup HAProxy as load balancers for gRPC with Cisco gear #

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
apt install haproxy




