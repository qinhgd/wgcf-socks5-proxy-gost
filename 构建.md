#!/bin/sh
set -e

# ==============================================================================
# 脚本配置
# ==============================================================================
BEST_IP_FILE="/wgcf/best_ips.txt"
OPTIMIZE_INTERVAL=21600
WARP_CONNECT_TIMEOUT=5
BEST_IP_COUNT=20
HEALTH_CHECK_INTERVAL=60

# ==============================================================================
# 工具函数
# ==============================================================================
red() { echo -e "\033[31m\033[01m$1\033[0m"; }
green() { echo -e "\033[32m\033[01m$1\033[0m"; }
yellow() { echo -e "\033[33m\033[01m$1\033[0m"; }

archAffix(){
    case "$(uname -m)" in
        aarch64 | arm64 | armv8 ) echo 'arm64' ;;
        x86_64 | amd64 ) echo 'amd64' ;;
        * ) red "❌ 不支持的CPU架构: $(uname -m)" && exit 1 ;;
    esac
}

# ==============================================================================
# IP优选相关函数
# ==============================================================================
download_warp_tool_if_needed() {
    if [ -f "warp" ]; then chmod +x warp; return; fi
    local arch=$(archAffix)
    local url="https://gitlab.com/Misaka-blog/warp-script/-/raw/main/files/warp-yxip/warp-linux-$arch"
    green "🌐 正在下载 WARP 优选工具 (架构: $arch)..."
    if curl -fsSL "$url" -o warp; then chmod +x warp; green "✅ WARP 优选工具下载完成。"; else red "❌ WARP 优选工具下载失败。" && exit 1; fi
}

run_ip_selection() {
    local ip_version_flag=""; [ "$1" = "-6" ] && ip_version_flag="-ipv6"
    green "🚀 开始优选 WARP Endpoint IP..."; download_warp_tool_if_needed
    ./warp -t "$WARP_CONNECT_TIMEOUT" ${ip_version_flag} > /dev/null
    if [ -f "result.csv" ]; then
        green "✅ 优选完成，正在处理结果..."
        awk -F, '($2+0) < 50 && $3!="timeout ms" {print $1}' result.csv | head -n "$BEST_IP_COUNT" > "$BEST_IP_FILE"
        if [ -s "$BEST_IP_FILE" ]; then green "✅ 已生成包含 $(wc -l < "$BEST_IP_FILE") 个IP的优选列表。"; else red "⚠️ 未能筛选出合适的IP，将使用默认地址。"; echo "engage.cloudflareclient.com:2408" > "$BEST_IP_FILE"; fi
        rm -f result.csv
    else
        red "⚠️ 未生成优选结果，将使用默认地址。"; echo "engage.cloudflareclient.com:2408" > "$BEST_IP_FILE"
    fi
}

# ==============================================================================
# 代理和连接核心功能
# ==============================================================================
_downwgcf() {
    yellow "正在清理 WireGuard 接口..."; wg-quick down wgcf >/dev/null 2>&1 || echo "wgcf 接口不存在或已关闭。"; yellow "清理完成。"; exit 0
}

update_wg_endpoint() {
    if [ ! -s "$BEST_IP_FILE" ]; then red "❌ 优选IP列表为空！将执行一次紧急IP优选..."; run_ip_selection "$1"; fi
    local random_ip=$(shuf -n 1 "$BEST_IP_FILE")
    green "🔄 已从优选列表随机选择新 Endpoint: $random_ip"
    sed -i "s/^Endpoint = .*$/Endpoint = $random_ip/" /etc/wireguard/wgcf.conf
}

# ---【已升级为全功能版】---
_startProxyServices() {
    if ! pgrep -f "gost" > /dev/null; then
        yellow "starting GOST proxy services..."
        
        local GOST_COMMAND="gost"
        
        # --- SOCKS5 代理配置 ---
        local SOCKS5_PORT="${PORT:-1080}"
        local AUTH_INFO=""
        [ -n "$USER" ] && [ -n "$PASSWORD" ] && AUTH_INFO="${USER}:${PASSWORD}@"
        local HOST_IP="${HOST:-0.0.0.0}"
        
        local SOCKS5_LISTEN_ADDR="socks5://${AUTH_INFO}${HOST_IP}:${SOCKS5_PORT}"
        GOST_COMMAND="${GOST_COMMAND} -L ${SOCKS5_LISTEN_ADDR}"
        green "✅ SOCKS5 代理已配置 (端口: ${SOCKS5_PORT})。"
        
        # --- HTTP 代理配置 (可选) ---
        # 如果设置了 HTTP_PORT 环境变量，则添加 HTTP 代理
        if [ -n "$HTTP_PORT" ]; then
            local HTTP_LISTEN_ADDR="http://${AUTH_INFO}${HOST_IP}:${HTTP_PORT}"
            GOST_COMMAND="${GOST_COMMAND} -L ${HTTP_LISTEN_ADDR}"
            green "✅ HTTP 代理已配置 (端口: ${HTTP_PORT})。"
        fi
        
        # 启动 gost
        eval "${GOST_COMMAND} &"
        
        yellow "✅ GOST 服务已启动。"
    fi
}
# ---【修改结束】---

_check_connection() {
    local check_url="ipinfo.io/json"; local curl_opts="--max-time 4 --silent"
    if [ "$1" = "-6" ]; then check_url="ipv6.google.com"; curl_opts="-6 --max-time 4 --silent"; fi
    if curl ${curl_opts} ${check_url} > /dev/null; then return 0; else return 1; fi
}

# ==============================================================================
# 主运行函数
# ==============================================================================
runwgcf() {
    trap '_downwgcf' ERR TERM INT
    [ ! -e "wgcf-account.toml" ] && wgcf register --accept-tos
    [ ! -e "wgcf-profile.conf" ] && wgcf generate
    cp wgcf-profile.conf /etc/wireguard/wgcf.conf
    [ "$1" = "-6" ] && sed -i 's/AllowedIPs = 0.0.0.0\/0/#AllowedIPs = 0.0.0.0\/0/' /etc/wireguard/wgcf.conf
    [ "$1" = "-4" ] && sed -i 's/AllowedIPs = ::\/0/#AllowedIPs = ::\/0/' /etc/wireguard/wgcf.conf
    [ ! -f "$BEST_IP_FILE" ] && run_ip_selection "$@"
    ( while true; do sleep "$OPTIMIZE_INTERVAL"; yellow "🔄 [定时任务] 开始更新IP列表..."; wg-quick down wgcf >/dev/null 2>&1 || true; run_ip_selection "$@"; yellow "🔄 [定时任务] IP列表更新完成。"; done ) &
    while true; do
        while true; do
            update_wg_endpoint "$@"; wg-quick up wgcf
            if _check_connection "$@"; then green "✅ WireGuard 连接成功！"; break; else red "❌ 连接失败，正在更换IP重试..."; wg-quick down wgcf >/dev/null 2>&1 || true; sleep 3; fi
        done
        _startProxyServices
        green "进入连接监控模式..."
        while true; do
            sleep "$HEALTH_CHECK_INTERVAL"
            if ! _check_connection "$@"; then red "💔 连接已断开！将返回连接阶段进行自动重连..."; wg-quick down wgcf >/dev/null 2>&1 || true; break; fi
        done
    done
}

# ==============================================================================
# 脚本入口
# ==============================================================================
cd /wgcf
if [ -z "$@" ] || [[ "$1" = -* ]]; then
    runwgcf "$@"
else
    exec "$@"
fi
