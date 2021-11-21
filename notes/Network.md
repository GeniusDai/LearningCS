# Network
*Computer Network -- Protocol Stack*

Port - IP - MAC
---------------

Segment / Package / Frame

在以太网交换机与路由器转发过程中，目的IP地址、目的MAC地址都可能被修改，前者根据子网地址，后者根据ARP缓存表

DHCP协议
* DHCP用于端系统自动获取IP地址，子网内必须至少要有一个DHCP服务器
* DHCP Server Port: 67, DHCP Client Port: 68
* DHCP基于UDP协议
* DHCP Request: 0.0.0.0:68 --> 255.255.255.255:67


链路层
-----

链路层是软硬件共同实现的，通过网卡中的芯片--链路层控制器提供支持。最初集成于PCI插槽，后来是主板。

链路层协议：以太网协议、WiFi、ARP。

名词解释
-------

DHCP: Dynamic Host Configuration Protocol

ARP: Address Resolution Protocol

MAC: Medium Access Control

NIC: Network Interface Card
