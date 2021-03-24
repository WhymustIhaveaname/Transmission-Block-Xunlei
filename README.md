# Transmission-Block-Xunlei
通过 transmission-remote 获取对端信息之后通过 iptables 限制迅雷的速度

### 为什么要限速迅雷？
经过观察，我发现迅雷客户端只下载不上传，或者说迅雷客户端从所有人那里下载但是只上传给迅雷而我使用的是 Transmission。P2P的精神在于分享，在于人人为我我为人人。今天我把文件分享给别人，是因为将来他们会把文件分享给更多的人。迅雷只从我这里下载却不上传，就是违背了P2P社区的共约，就是抛弃了我们的诺言，就是白嫖。所以我要限制迅雷客户端的速度。

### 简单版脚本

这个最简化的脚本解释了基本原理。

```
username="yourusername"
password="yourpassword"
host="127.0.0.1"
port=9091
chain="OUTPUT"
speedlimit=15 # speed = ($speedlimit+5)x1.5 (kb/s)

ips=`transmission-remote $host:$port --auth $username:$password -t all --info-peers`
echo "$ips"

for client in Xunlei Thunder
do
    echo -n "dealing $client: "
    # echo $ips 没有换行符，加引号才有
    for i in `echo "$ips" | grep $client | cut --delimiter " " --fields 1`
    do
        echo $i
        iptables -A $chain -m limit -d $i --limit $speedlimit/s -j ACCEPT
        iptables -A $chain -d $i -j DROP
    done
done
```

### 真正使用的脚本

这是我真正在用的脚本，它

* 会判断规则是否在 iptables 中，不在才添加
* 每到整30分钟清空限速名单重新添加
* 简化了输出

我在 crontab 中让它每两分钟执行：`*/2 * * * * cd /home/blabla && ./block_xunlei.sh 1>>block_xunlei.log 2>&1`。

```
date
username="yourusername"
password="yourpassword"
host="127.0.0.1"
port=9091
chain="OUTPUT"
speedlimit=15 # speed = ($speedlimit+5)x1.5 (kb/s)

ips=`transmission-remote $host:$port --auth $username:$password -t all --info-peers`
#echo "$ips"

minute=$(date "+%M")
if [[ $minute == 30 ]]
then
    echo clear chain $chain
    iptables -F $chain
fi

rules=`iptables -nL $chain`
#echo "$rules"

for client in Xunlei Thunder
do
    echo -n "dealing $client: "
    # echo $ips 没有换行符，加引号才有
    for i in `echo "$ips" | grep $client | cut --delimiter " " --fields 1`
    do
        if [[ $rules =~ $i ]]
        then
            echo -n "$i, "
        else
            echo -n "$i NOT in rules, "
            iptables -A $chain -m limit -d $i --limit $speedlimit/s -j ACCEPT
            iptables -A $chain -d $i -j DROP
        fi
    done
    echo ""
done
```
