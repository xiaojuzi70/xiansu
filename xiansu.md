#!/bin/bash

# 配置
LIMITED_FILE="/tmp/traffic_limited"

# 获取网卡名称
echo "🔍 检测到以下网卡："
ip -o link show | awk -F': ' '{print $2}'
read -p "请输入要监控的网卡名称: " NIC

# 设定流量上限
read -p "📊 请输入每月允许的最大流量（单位：GB）: " LIMIT_GB
LIMIT_MB=$((LIMIT_GB * 1024))

# 设定限速规则
read -p "🚀 超过 90% 后限速（单位 Mbps，默认 5）: " LIMIT_90
LIMIT_90=${LIMIT_90:-5}
read -p "🚨 超过 100% 后限速（单位 Mbps，默认 1）: " LIMIT_100
LIMIT_100=${LIMIT_100:-1}

# 设定重置周期
read -p "🔄 多少天后自动重置流量（默认 30 天）: " RESET_DAYS
RESET_DAYS=${RESET_DAYS:-30}

# 安装依赖
echo "⏳ 安装依赖..."
apt update && apt install -y vnstat iproute2
vnstat --create -i $NIC 2>/dev/null
systemctl enable vnstat
systemctl restart vnstat

# 设置定时任务：每 5 分钟检测流量
(crontab -l 2>/dev/null | grep -v "check_bandwidth.sh"; echo "*/5 * * * * /root/bandwidth_limiter.sh check $NIC $LIMIT_MB $LIMIT_90 $LIMIT_100") | crontab -

# 设置定时任务：每 X 天重置流量
(crontab -l 2>/dev/null | grep -v "vnstat --reset"; echo "0 0 */$RESET_DAYS * * vnstat --reset") | crontab -

# 生成流量监控脚本
cat > /root/bandwidth_limiter.sh << 'EOF'
#!/bin/bash

if [ "$1" == "check" ]; then
    NIC=$2
    LIMIT_MB=$3
    LIMIT_90=$4
    LIMIT_100=$5
    LIMITED_FILE="/tmp/traffic_limited"

    # 获取流量
    USAGE=$(vnstat --dumpdb | grep "m;" | awk -F';' '{print $4 + $5}')
    REMAINING_MB=$((LIMIT_MB - USAGE))
    REMAINING_GB=$((REMAINING_MB / 1024))

    # 根据使用情况动态限速
    if [[ "$USAGE" -ge "$LIMIT_MB" ]]; then
        LIMIT_SPEED_KBPS=$((LIMIT_100 * 1024))  # 超限 -> 1 Mbps
        echo "🚨 流量超限，限速 ${LIMIT_100} Mbps"
    elif [[ "$USAGE" -ge "$((LIMIT_MB * 90 / 100))" ]]; then
        LIMIT_SPEED_KBPS=$((LIMIT_90 * 1024))  # 90% -> 5 Mbps
        echo "⚠️ 已使用 90% 以上流量，限速 ${LIMIT_90} Mbps"
    else
        LIMIT_SPEED_KBPS=0
        echo "✅ 流量正常，无需限速"
    fi

    # 应用限速
    if [[ "$LIMIT_SPEED_KBPS" -gt 0 ]]; then
        if [[ ! -f "$LIMITED_FILE" ]]; then
            tc qdisc add dev "$NIC" root tbf rate ${LIMIT_SPEED_KBPS}kbps burst 16000 limit 30000
            touch "$LIMITED_FILE"
        fi
    else
        if [[ -f "$LIMITED_FILE" ]]; then
            tc qdisc del dev "$NIC" root
            rm -f "$LIMITED_FILE"
        fi
    fi

    # 显示剩余流量
    echo "📊 已使用流量: ${USAGE}MB / ${LIMIT_MB}MB"
    echo "📉 剩余流量: ${REMAINING_GB}GB"

elif [ "$1" == "unlimit" ]; then
    NIC=$(ip -o link show | awk -F': ' '{print $2}' | head -n 1)  # 自动获取网卡
    LIMITED_FILE="/tmp/traffic_limited"

    echo "🔓 解除 ${NIC} 的限速..."
    tc qdisc del dev "$NIC" root
    rm -f "$LIMITED_FILE"
    echo "✅ 限速已解除"

else
    echo "❌ 无效的参数，请使用 'check' 或 'unlimit'."
fi
EOF

chmod +x /root/bandwidth_limiter.sh

# 提示安装完成
echo "✅ 安装完成！"
echo "📊 使用 'vnstat -m' 查看流量"
echo "🔍 手动检测流量: 'bash /root/bandwidth_limiter.sh check'"
echo "🚀 解除限速: 'bash /root/bandwidth_limiter.sh unlimit'"
