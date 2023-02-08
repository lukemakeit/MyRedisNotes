## tcpdump

tcpdump 和 wireshark 是最常用的网络抓包和分析工具。

wireshark除了抓包外,还提供可视化网络包的图形页面。

tcpdump的命令如: `tcpdump -i eth1 icmp and host 183.232.231.174 -nn`

- `-i  eth1` 表示抓取`eth1`网口的数据包;
- `icmp` 表示抓取icmp协议的数据报;
- `host`:表示主机过滤,抓取对应IP的数据包;
- `-nn`: 表示不解析ip地址和端口号的命令;

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220222153529086.png" alt="image-20220222153529086" style="zoom:50%;" />

<img src="https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220222153611212.png" alt="image-20220222153611212" style="zoom:50%;" />

tcpdump抓取的包，通过`-w`参数保存成`.pcap`后缀的文件，接着用`wireshark`工具进行数据包分析。

命令: `tcpdump -i eth1 tcp and host 11.152.231.151 and port 50009 -w redis-50009.pcap`

![image-20220222155905078](https://my-typora-pictures-1252258460.cos.ap-guangzhou.myqcloud.com/img/image-20220222155905078.png)

