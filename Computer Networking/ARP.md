# ARP

地址解析协议

根据IP地址获取物理地址。

解析过程如下：

- 在自己的ARP缓冲区建立一个ARP列表，以表示IP地址和MAC地址之间的对应关系
- 当存在没有映射的ip地址式，向同一子网的所有主机发送ARP数据包（源主机的IP和MAC地址）
- 本网络的所有主机收到该ARP数据包时，若请求的自己的ip地址，则将IP与MAC地址返回数据包，必要的话顺便更新自己的ARP列表
- 源主机收到结果，写入ARP列表，并利用此信息发送数据
