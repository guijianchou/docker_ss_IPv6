Docker下安装ss后关于IPv6的连通性折腾：
参考资料和链接如下：
1.chagpt
2.Enable IPv6 support for shadowsocks in docker：https://tonybuaa.github.io/zh-CN/2019/01/08/Enable-IPv6-support-for-ss-in-docker/
3.Enable IPv6 support：https://docs.docker.com/config/daemon/ipv6/
4.https://v2ex.com/t/641022


1.以SS为基础遇到的各种问题和基本思路：
当我按照docker官网配置完IPv6后：
修改/etc/docker/daemon.json
{
  "ipv6": true,
  "fixed-cidr-v6": "2001:db8:1::/64"
}
然后重启systemctl reload docker

2.当查看IPV6：：确实在docker的container容器内能正常获取到IPv6地址，然是当我ping时却ping不通：
docker run --network=bridge --rm -it busybox ping -6 -c4 google.com
PING google.com (2607:f8b0:4005:801::200e): 56 data bytes
--- google.com ping statistics ---
4 packets transmitted, 0 packets received, 100% packet loss

3.我的基本思路都放在了防火墙上，然后就做了如下尝试：
sudo ip6tables -I FORWARD -i docker0 -o eth0 -j ACCEPT
sudo ip6tables -I FORWARD -i eth0 -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
sudo ip6tables -I INPUT -i docker0 -j ACCEPT
sudo ip6tables -I OUTPUT -o docker0 -j ACCEPT
但是依旧不通。于是就开始chatgpt寻找解决方案》
chagpt给出了如下建议：
（1）确认宿主机已经开启 IPv6 功能，并且网络连接正常。
（2）检查容器的网络配置是否正确，包括网络接口、IPv6 地址和网关等参数是否设置正确。
（3）检查 iptables 规则是否正确设置，可以通过以下命令查看当前规则：sudo ip6tables -L

4.根据chatgpt的建议，挨个排除。最终在3上找到了问题所在：
如果使用了 Docker 的 NAT 方式，需要确保正确设置 iptables 规则，以允许容器内部和容器与主机之间进行 IPv6 通信。可以使用以下命令添加规则：
sudo ip6tables -t nat -A POSTROUTING -s <container-ipv6-address> ! -o docker0 -j MASQUERADE
其中 <container-ipv6-address> 是容器的 IPv6 地址。该规则将在 NAT 表中添加一条记录，以便在从容器发送 IPv6 数据包时自动转发数据包并修改源地址为主机的 IPv6 地址。

而如果设置按照docekr官方的{"ipv6": true,"fixed-cidr-v6": "2001:db8:1::/64"}，则执行以下代码：
ip6tables -t nat -A POSTROUTING -s 2001:db8:1::/64 -j MASQUERADE

然后测试：
docker run --network=bridge --rm -it busybox ping -6 -c4 google.com
PING google.com (2607:f8b0:4005:801::200e): 56 data bytes
64 bytes from 2607:f8b0:4005:801::200e: seq=0 ttl=59 time=3.533 ms
64 bytes from 2607:f8b0:4005:801::200e: seq=1 ttl=59 time=1.752 ms
64 bytes from 2607:f8b0:4005:801::200e: seq=2 ttl=59 time=1.907 ms
64 bytes from 2607:f8b0:4005:801::200e: seq=3 ttl=59 time=1.743 ms

--- google.com ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss

