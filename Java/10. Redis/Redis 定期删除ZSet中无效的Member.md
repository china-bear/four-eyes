如果使用redis zset结构，key不能直接设置过期时间，又想对member设置过期删除时，可以使用这个脚本根据score进行过期删除。大家可以参考下，score的格式需要自定义调整下。

删除脚本redis-del-keys.sh (main)

```powershell
#!/bin/bash
##redis主机IP
host=$1
##redis端口
port=$2
##redis端口
password=$3
##redis db
db=$4
##key模式
pattern=$5
##游标
cursor=0
##退出信号
signal=0
##将一周前此时刻转换为时间戳，精确到毫秒
end=`date "+%Y%m%d%H%M" -d '7 day ago'`
##将一周前此时刻转换为时间戳，精确到毫秒
start=`date "+%Y%m%d%H%M" -d '1000 day ago'`
echo $end


##循环获取key并删除
while [ $signal -ne 1 ]
    do
        echo "cursor:${cursor}"
        sleep 2
        echo $host
        ##将redis scan得到的结果赋值到变量
        re=$(redis-cli -h $host -p $port -a $password -n $db -c  scan $cursor count 1000 match $pattern)
        echo $port
        ##以换行作为分隔符
        IFS=$'\n' 
        #echo $re
        echo 'arr=>'
        ##转成数组
        arr=($re)
        ##打印数组长度
        echo 'len:'${#arr[@]}
        ##第一个元素是游标值
        cursor=${arr[0]}
        ##游标为0表示没有key了
        if [ $cursor -eq 0 ];then
            signal=1
        fi
        ##循环数组
    for key in ${arr[@]}
        do
            ##echo $key
            if [ $key != $cursor ];then
                echo "key:"$key
                ##删除key
                redis-cli -h $host -p $port -a $password -n $db -c zremrangebyscore $key $start $end >/dev/null  2>&1
            fi
    done
done
echo 'done'
```

时间转换

```powershell
timeformat=$1
before=`date "+%Y-%m-%d %H:%M:%S" -d '7 day ago'`  
timeStamp=`date -d "$before" +%s`   
#将current转换为时间戳，精确到毫秒  
endtime=$((timeStamp*1000+`date "+%N"`/1000000)) 

if [ "$timeformat" = "minute" ]; then
	#将current转换为时间戳，精确到毫秒
	endtime=$(date "+%Y%m%d%H%M" -d '7 day ago')
fi
echo $endtime
```

脚本如上

linux定时调度配置
chmod 744 /app/redis-4.0.8/sh/redis-del-keys.sh
crontab -e

## 每天凌晨两点调度
0 2 * * * /app/redis-4.0.8/sh/redis-del-keys.sh

使用示例
sh redis-del-keys.sh local 6379 12345 5 userid:*

其中‘redis-del-keys.sh’为删除脚本(main), 'local 6379 12345' 分别为host、port、password， ‘5’为db index， ‘userid:*’为key的前缀
