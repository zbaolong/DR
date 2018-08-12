# DR
DR production
111111111111111111111111111111111111
#!/bin/sh

LANG=en_US.UTF-8
CPU_USAGE_THRESHOLD=90
MEM_USAGE_THRESHOLD=90
DISK_USAGE_THRESHOLD=70
DROP_RATE_THRESHOLD=1/1000000

GREEN=""
RED=""
ENDC=""

ERR="ERROR"
function output
{
    key=$1
    val=$2
    if [[ -n $val ]];then
        echo -e "$key:\n$val"
    else
        echo "-------------------------$key-------------------------"

    fi
}

function _log_succ()
{
    typeset message=${1?"need messages"}

    echo -e "${message}"

    return 0
}

function _log_err()
{
    typeset message=${1?"need messages"}

    echo -e "$ERR: ${message}${ENDC}"

    return 0
}

function net_io_drop_rate()
{
    nic=${1?"Need a nic"}
    interval=${2:-"10"}
    per_unit=${3:-"1000000"}

    net_proc_file="/proc/net/dev"
    net_io_stats=$(egrep "$nic" ${net_proc_file})
    R_packets=$(echo $net_io_stats | awk '{print $3}')
    R_drop=$(echo $net_io_stats | awk '{print $5}')
    T_packets=$(echo $net_io_stats | awk '{print $11}')
    T_drop=$(echo $net_io_stats | awk '{print $13}')
    
    sleep $interval
    net_io_stats_next=$(egrep "$nic" ${net_proc_file})
    R_packets_next=$(echo $net_io_stats_next | awk '{print $3}')
    R_drop_next=$(echo $net_io_stats_next | awk '{print $5}')
    T_packets_next=$(echo $net_io_stats_next | awk '{print $11}')
    T_drop_next=$(echo $net_io_stats_next | awk '{print $13}')

    if ((R_packets_next == R_packets));then
        R_drop_per=0
    else
        R_drop_per=$(echo | awk '{ print ('$R_drop_next'-'$R_drop')*'$per_unit'/('$R_packets_next'-'$R_packets')}')
    fi

    if (( T_packets_next == T_packets));then
        T_drop_per=0
    else
        T_drop_per=$(echo | awk '{ print ('$T_drop_next'-'$T_drop')*'$per_unit'/('$T_packets_next'-'$T_packets')}')
    fi

    echo "R_drop_per=$R_drop_per,T_drop_per=$T_drop_per"

    return 0
}

CPU_USAGE=$(top -b -n 2 | grep '%Cpu(s)' | tail -n 1)
CPU_UESD=$(echo ${CPU_USAGE} | awk -F',' '{print $4}' | awk '{print 100-$1}')
CPU_FLAG=$(echo | awk '{ if('$CPU_UESD' <= '$CPU_USAGE_THRESHOLD') print "true" }')

output "操作系统性能巡检结果"
output "CPU使用率" "$CPU_USAGE"
if [[ $CPU_FLAG == "true" ]];then
    _log_succ "CPU使用率正常, used ${CPU_UESD}%\n"
else
    _log_err "CPU使用率超过阈值${CPU_USAGE_THRESHOLD}%, used ${CPU_UESD}%\n"
fi

MEM_USAGE=$(free -k | awk '{
                        if(NR==2){
                        printf "%-15s%10s MB\n", "Total Memory:", $2/1024; total = $2;}
                        if(NR==3){
                        printf "%-15s%10s MB\n", "Used Memory:", $3/1024; used = $3;
                        printf "%-15s%10s %\n", "Used Percent:", used*100.0/total;}
                        if(NR==4){
                        printf "%-15s%10s MB\n", "Paging Space:", $2/1024;
                        printf "%-15s%10s MB\n", "Used Paging:", $3/1024;
                        printf "%-15s%10s %\n", "Used Percent:", $3*100.0/$2; }}')


output "内存使用情况" "$MEM_USAGE"
MEM_FLAG=$(echo "$MEM_USAGE" | grep 'Used Percent' | awk '{ if($3 >= '$MEM_USAGE_THRESHOLD')print $3}')

if [ -z "$MEM_FLAG" ];then
    _log_succ "内存用率正常\n" 
else
    _log_err "内存使用率超过阈值${MEM_USAGE_THRESHOLD}%, used ${MEM_FLAG}%\n"
fi

PARTITION_USAGE=$(df -lhTP)
PARTITION_USED=$(echo "$PARTITION_USAGE" | awk '{if(NR>1) print $6$7}')

output "磁盘分区使用情况" "$PARTITION_USAGE"
PARTITION_FLAG=$(echo "$PARTITION_USED" | awk -F'%' '{if($1 >= '$DISK_USAGE_THRESHOLD')print $2" "$1"%"}')
if [ -z "$PARTITION_FLAG" ];then
    _log_succ "磁盘分区使用率正常\n" 
else
    _log_err "如下磁盘分区使用率超过阈值${DISK_USAGE_THRESHOLD}%\n${PARTITION_FLAG}\n"
fi


linked_nics=$(for i in `ip link sh |grep '^[1-9]' |awk -F": " '{if($2 != "lo")print $2}'`; do if [[ `ethtool $i | grep "Link detected" |grep yes` != "" ]] ; then ifconfig $i | head -1 | awk -F: '{print $1}'; fi; done)


echo "网络接口使用情况"
for nic in $linked_nics; do
    drop_per=$(net_io_drop_rate "$nic" "5" ${DROP_RATE_THRESHOLD##*/})
    R_drop_per=$(echo $drop_per | awk -F',' '{print $1}'  | awk -F'=' '{print $2}')
    T_drop_per=$(echo $drop_per | awk -F',' '{print $2}'  | awk -F'=' '{print $2}')

    if $(echo $R_drop_per | awk '{ if($1 >'${DROP_RATE_THRESHOLD%%/*}') print "false"; else print "true" }');then
        _log_succ "$nic: 收包丢包率 $R_drop_per/${DROP_RATE_THRESHOLD##*/}"
    else
        _log_err "$nic: 收包丢包率 $R_drop_per/${DROP_RATE_THRESHOLD##*/}"
    fi

    if $(echo $T_drop_per | awk '{ if($1 >'${DROP_RATE_THRESHOLD%%/*}') print "false"; else print "true" }');then
        _log_succ "$nic: 发包丢包率 $T_drop_per/${DROP_RATE_THRESHOLD##*/}"
    else
        _log_err "$nic: 发包丢包率 $T_drop_per/${DROP_RATE_THRESHOLD##*/}"
    fi
	
done



22222222222222222222222222222222222222222222222222:
#!/usr/bin/ksh

LANG=en_US.UTF-8

echo "设备信息$(uname -uM)"
echo " "

echo "================cpu状态================="

if [ 90 -le "$(sar 1 10|grep Average|awk '{print $5}')" ]
then
    echo "cpu使用率低于10%，正常！"
else
    echo "cpu使用率高于10%，异常"
fi

echo "================内存状态================="

if [ "$(expr $(svmon -G|grep memory|awk '{print $3/$2}') \> 0.6 )" -eq 0 ]
then
    echo "内存使用率低于60%，正常!"
else
    echo "内存使用率高于60%，异常!"
fi

echo "============交换内存空间使用============="
if [ 70 -ge "$(lsps -a|awk 'NR==2{print$5}')" ]
then
    echo "交换内存空间使用率低于70%，正常！"
else
    echo "交换内存空间使用率高于70%，异常"
fi

echo "================硬盘使用================="

if [ 80 -le "$(df -g|grep -w "/"|awk '{print $4*1}')" ]
then
    echo "1. /目录使用超过80%，异常!"
else
    echo "1. /目录使用低于80%，正常!"
fi

if [ 80 -le "$(df -g|grep -w "/home"|awk '{print $4*1}')" ]
then
    echo "2. /home目录使用超过80%，异常!"
else
    echo "2. /home目录使用低于80%，正常!"
fi

if [ 80 -le "$(df -g|grep -w "/tmp"|awk '{print $4*1}')" ]
then
    echo "3. /tmp目录使用超过80%，异常!"
else
    echo "3. /tmp目录使用低于80%，正常!"
fi

echo "================错误报告================="
if [ -n "$(errpt -d H -T PERM|grep "UNDETERMINED ERROR")" ]
then
    echo "存在硬件错误报告，打印最近5行"
	echo "$(errpt -d H -T PERM |awk 'NR>1'|head -n 5)"
else
    echo "不存在硬件错误报告，正常！"
fi

if [ -n "$(errpt -d S -T PERM|grep "SOFTWARE PROGRAM ERROR")" ]
then
    echo "存在软件错误报告，打印最近5行"
	echo "$(errpt -d S -T PERM |awk 'NR>1'|head -n 5)"
else
    echo "不存在软件错误报告，正常！"
fi
