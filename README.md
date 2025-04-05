# sni-proxy

This tool can intercept connections on HTTP/HTTPS ports and redirect them to an HTTP proxy using the CONNECT method.


```
|                  IP                   |           TLS           |
|---------------------------------------|-------------------------|
| src_ip | src_port | dst_ip | dst_port | ... | server_name | ... |
=>
TCP: proxy_host:proxy_port
    CONNECT server_name:dst_port
    X-Forwarded-For: src_ip


|                  IP                   |                       HTTP                      |
|---------------------------------------|-------------------------------------------------|
| src_ip | src_port | dst_ip | dst_port | GET / HTTP/1.1                                  |
|        |          |        |          | Host: server_name:server_port                   |
=>
TCP: proxy_host:proxy_port
    CONNECT server_name:server_port
    X-Forwarded-For: src_ip

```
For HTTPS, it only works with TLS and only with unencrypted SNI (version 1.3 supports ESNI—such connections will be dropped).

For HTTP, it only works if the Host header is present (HTTP/1.0 does not use the Host header).

It always passes the domain from SNI (or from the Host header for HTTP) to the proxy, completely ignoring the original destination IP.

    For HTTPS, the proxy receives the same port as in the original request.

    For HTTP, the port is extracted from the Host header.

It sends the X-Forwarded-For header to the proxy (useful, for example, when intercepting all VM traffic and routing it through a proxy—the proxy will see the correct source VM). The proxy must be configured to trust headers from allowed hosts.

It supports NO_PROXY, but it's highly recommended to exclude unnecessary traffic via iptables rules instead, as this feature can slow things down and potentially break.


```
apt-get install sni-proxy
vim /etc/default/sni-proxy
```
```
iptables -t nat -I OUTPUT -p tcp --dport 443 ! -d 127.0.0.0/8 -m owner ! --uid-owner $(getent passwd proxy | cut -d ":" -f 3) -j DNAT --to 127.0.0.1:3130
iptables -t nat -I OUTPUT -p tcp --dport 80 ! -d 127.0.0.0/8 -m owner ! --uid-owner $(getent passwd proxy | cut -d ":" -f 3) -j DNAT --to 127.0.0.1:3131
```
or
```
# https
iptables -t nat -A OUTPUT -p tcp --dport 443 -d 127.0.0.0/8 -j ACCEPT
iptables -t nat -A OUTPUT -p tcp --dport 443 -d 10.0.0.0/8 -j ACCEPT
iptables -t nat -A OUTPUT -p tcp --dport 443 -d 192.168.0.0/16 -j ACCEPT
iptables -t nat -A OUTPUT -p tcp --dport 443 -m owner ! --uid-owner 13 -j DNAT --to-destination 127.0.0.1:3130
# http
iptables -t nat -A OUTPUT -p tcp --dport 80 -d 127.0.0.0/8 -j ACCEPT
iptables -t nat -A OUTPUT -p tcp --dport 80 -d 10.0.0.0/8 -j ACCEPT
iptables -t nat -A OUTPUT -p tcp --dport 80 -d 192.168.0.0/16 -j ACCEPT
iptables -t nat -A OUTPUT -p tcp --dport 80 -m owner ! --uid-owner 13 -j DNAT --to-destination 127.0.0.1:3131
```

For FORWARD trafik:
```
iptables -t nat -I POSTROUTING -p tcp --dport 443 -j DNAT --to 127.0.0.1:3130
iptables -t nat -I POSTROUTING -p tcp --dport 80 -j DNAT --to 127.0.0.1:3131
```

Logs:
```
journalctl -u sni-proxy -f
```

