# Secure Cluster

## Firewall
!> THIS IS **NOT TESTED**

You can use `ipset` to create a network set that contains all the
IPs of your cluster nodes and use it to setup `iptables` rules.

### Create ipset
!> THIS IS **NOT TESTED**

```bash
# install ipset
apt install ipset
# create an iphash type of set
ipset -N my-k8s-cluser iphash
# add node with IP 1.2.3.4
ipset -A my-k8s-cluster 1.2.3.4
# add node with IP 5.6.7.8
ipset -A my-k8s-cluster 5.6.7.8
```

### Create rules
!> THIS IS **NOT TESTED**

```bash
# allow everything from local machine
iptables -A INPUT -i lo -j ACCEPT
# allow everything from established connections
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# allow ping
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
# allow SSH and WEB ports
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
# allow access to kubernetes ports from the cluster set
iptables -A INPUT -p tcp --dport 30000 -m set --match-set my-k8s-cluser src -j ACCEPT
iptables -A INPUT -p tcp --dport 30000 -j DROP
# block everything else
iptables -A INPUT -j DROP
# inspect rules
iptables -S
```

?> Check [required ports](https://kubernetes.io/docs/setup/independent/install-kubeadm/#check-required-ports)
?> for more details on which ports are needed by `kubernetes`

### Persist rules
!> THIS IS **NOT TESTED**

Use `iptables-persistent`

```bash
apt install iptables-persistent
iptables-save > /etc/iptables/rules.v4
iptables-save > /etc/iptables/rules.v6
```
