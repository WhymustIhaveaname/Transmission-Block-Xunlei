# Transmission-Block-Xunlei
## 通过 transmission-remote 获取对端信息之后通过 iptables 限制迅雷的速度

### 为什么要限速迅雷？
经过观察，我发现迅雷客户端只下载不上传，或者说迅雷客户端从所有人那里下载但是只上传给迅雷而我使用的是 Transmission。P2P的精神在于分享，在于人人为我我为人人。今天我把文件分享给别人，是因为将来他们会把文件分享给更多的人。迅雷只从我这里下载却不上传，就是违背了P2P社区的公约，就是抛弃了我们的诺言，就是白嫖。所以我要限制迅雷客户端的速度。

### 简单版脚本

这个最简化的脚本解释了基本原理。网上有建议说直接开启强制加密就可以屏蔽大部分迅雷，但是我觉得这不好，第一是迅雷也会升级现在也有迅雷会用加密了，第二是我发现大量的 Transmission 和其他开源客户端都不加密，强制开启加密会把它们也过滤掉。所以最终，我决定采用我的方法：通过 transmission-remote 获取对端信息之后通过 iptables 限制迅雷的速度。

```
#!/bin/bash
ips=`transmission-remote 127.0.0.1:9091 --auth yourusername:yourpassword -t all --info-peers`
echo "$ips"

for client in xunlei thunder gt0002 xl0012 xfplay dandanplay dl3760 qq
do
    echo -n "dealing $client: "
    # echo $ips 没有换行符，加引号才有
    for i in `echo "$ips" | grep $client | cut --delimiter " " --fields 1`
    do
        echo $i
        # 下面两句话的意思是，每秒钟的前 5 个包和之后的 15 个包会被 accept，再多了就丢弃
        # 按每个包 1500 byte 算这就是限速到 30 kb/s
        iptables -I OUTPUT -d $i -j DROP
        iptables -I OUTPUT -m limit -d $i --limit 15/s --limit-burst 5 -j ACCEPT
    done
done
```

### 真正使用的脚本

这是我真正在用的脚本，它

* 会判断规则是否在 iptables 中，不在才添加
* 每到整4小时的整30分钟清空限速名单
* 增加了对 ipv6 的支持
* 简化了输出

我在 crontab 中让它每两分钟执行：`*/2 * * * * cd /home/blabla && ./block_xunlei.sh 1>>block_xunlei.log 2>&1`。注意由于 iptables 需要 sudo 权限，所以应当放置在 root 的 crontab 下。

另外，由于 crontab 运行时 PATH 为空，所以各种常见的应用程序它经常找不到 (感谢 @NeraSnow 的 Issue#1)。对此，有两种解决方法：

1. 在 crontab 中加入 `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin`，此后所有的 crontab 脚本都能找到各种应用程序。其中的 PATH 是在终端中用 `echo $PATH` 获得的。
2. 将脚本中的 iptables 和 ip6tables 改为绝对路径 `/usr/sbin/iptables` 和 `/usr/sbin/ip6tables`。

```
#!/bin/bash
date
username="yourusername"
password="yourpassword"
host="127.0.0.1"
port=9091
chain="OUTPUT"

# speed = ($speedlimit+$burstlimit)x1.5 (kb/s)
speedlimit=15
burstlimit=1

ips=`transmission-remote $host:$port --auth $username:$password -t all --info-peers`

minute=$(date "+%M")
hour=$(date "+%H")
if (( ${minute#0} == 30 && ${hour#0}%4 == 0 )); then
    # ${minute#0}是去掉开始的0, 否则bash会按8进制理解
    # (()), [[]] 分别是 [] 对数值运算和字符串运算的升级版
    echo clearing chain $chain
    iptables -F $chain
    ip6tables -F $chain
fi

rules=`iptables -nL $chain; ip6tables -nL $chain`

for client in xunlei thunder gt0002 xl0012 xfplay dandanplay dl3760 qq
do
    echo -n "dealing $client: "
    for i in `echo "$ips" | grep -i $client | cut --delimiter " " --fields 1`
    do
        if [[ $rules =~ $i ]]; then
            echo -n "$i, "
        else
            echo -n "$i NOT in rules, "
            if [[ $i =~ ":"  ]]; then # if this is an ipv6 address
                ip6tables -I $chain -d $i -j DROP
                ip6tables -I $chain -m limit -d $i --limit $speedlimit/s --limit-burst $burstlimit -j ACCEPT
            else
                iptables -I $chain -d $i -j DROP
                iptables -I $chain -m limit -d $i --limit $speedlimit/s --limit-burst $burstlimit -j ACCEPT
            fi
        fi
    done
    echo ""
done
```
