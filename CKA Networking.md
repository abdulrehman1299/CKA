 ![[Pasted image 20250104182946.png]]
**Switch** is being used to form a network. After adding the private IP addresses of the two servers, they are able to communicate with each other.
![[Pasted image 20250104183249.png]]
To allow connection between **two different networks**, **router** is used. Router has IPs of **both** networks.
To let the server **B** know that it can use router for external communication, **Route Tables** are used. To add a **route** in route table, use a **gateway**.
![[Pasted image 20250104183655.png]]
These routes will need to be entered in **all systems** of network A for communication with network B.
![[Pasted image 20250104184124.png]]
For access to internet to any site, we use the **default** destination. This means that if none of the rules are met then the **default** destination will be used. Instead of **default**, **0.0.0.0** can also be used.
To use a server as a **router**, we need to attach two network interfaces with it and add routes in each of the server's route tables to send message to router server for communication in another network.
To allow the **router server** to forward traffic received from one interface to another, we use **IP Forwarding**. This is done by editing the file mentioned below:
![[Pasted image 20250104191148.png]]

```
ip link (used to list down interfaces)
ip addr (used to see IP addresses attached to interfaces)
ip addr add (add IP address to an interface)
route (to see the route table)
ip route add (used to add entries into the routing table)
cat /proc/sys/net/ipv4/ip_forward (to check ip forwarding is on)
```

----------------------------
![[Pasted image 20250104193930.png]]
For host A to communicate with host B using the name **db**, we need to add the mapping in **/etc/hosts**. We can have multiple mappings for the same IP address.
To avoid storing the names of servers in host's personal /etc/hosts file, a central system is established using the **DNS Server**.

To allow host A to use DNS server for IP mapping. Add **nameserver** in **cat /etc/resolv.conf**
So, whenever any domain name not mentioned in **/etc/hosts** is searched, the system will automatically send request to nameserver.
![[Pasted image 20250104194717.png]]
We can use tools like **nslookup** and **dig** for getting information regarding domain names.
