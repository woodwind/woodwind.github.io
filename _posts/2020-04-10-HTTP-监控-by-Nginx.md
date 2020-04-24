## 技术组件
* prometheus
* grafana
* nginx

以上组件版本号自理

## 方案选型
nginx监控上报到prometheus一般有两种方式
1. [nginx-vts-module]("https://github.com/vozlt/nginx-module-vts")
2. [nginx-lua-prometheus]("https://github.com/knyar/nginx-lua-prometheus")

两者孰优孰劣最好自己体验下，这里说下我的体验感受。
* vts 通过module和conf进行监控配置，不能自由的设置label，适合不需要对监控数据做多维度筛选的场景，同时还有现成exporter和dashboard
* lua 通过调用prometheus的sdk自行统计，需要编写lua代码，但是可以随意定义label，所以业务扩展能力很强

考虑我的需求比较复杂，这里采用了lua的方式。因为vts的方式也走了一遍，有空的话可以梳理下。

## 准备工作
nginx + lua 或者 openresty，核心在于lua的支持

## 安装nginx-lua-prometheus
```
wget https://github.com/knyar/nginx-lua-prometheus/archive/0.20181120.tar.gz
tar -xzvf 0.20181120.tar.gz
```
解压后的prometheus.lua就是我们要的library

## 添加配置

### nginx http context
```
lua_shared_dict prometheus_metrics_dict 64M; # 共享内存的大小自定义
lua_package_path "/data/nginx_lua/?.lua;;"; # 我这里是为了后续统一管理lua文件，也可以按需单独使用

init_by_lua '
    prometheus_metrics = require("prometheus_metrics") # prometheus_metrics 是对prometheus的二次封装
    prometheus_metrics.init("prometheus_metrics_dict")
';

server {
    ...
    log_by_lua '
        prometheus_metrics.log()
    ';
    ...
}

server {
    listen 9145;

    location /metrics {
        prometheus_metrics.display()
    }
}
```

### prometheus_metrics
```
local prometheus
local metric_requests
local metric_request_latency
local metric_connection_rtt
local metric_connection_rtt_var
local metric_connections

local function init(dict_name)
    prometheus = require("prometheus").init(dict_name)
    metric_requests = prometheus:counter(
        "http_requests_total", "number of http requests", 
        {"host", "path", "status", "client", "version", "country"})
    metric_request_latency = prometheus:histogram(
        "http_request_duration_seconds", "http request latency", 
        {"host", "path", "country"},
        {0.005, 0.01, 0.03, 0.05, 0.1, 0.3, 0.5, 1, 3, 5})
    metric_connection_rtt = prometheus:histogram(
        "tcp_connection_rtt", "tcp rtt microseconds (μs)",
        {"country"},
        {50000, 100000, 200000, 300000, 400000, 500000, 1000000, 2000000, 3000000})
    metric_connection_rtt_var = prometheus:histogram(
        "tcp_connection_rtt_var", "tcp rttvar microseconds (μs)", 
        {"country"},
        {50000, 100000, 200000, 300000, 400000, 500000, 1000000, 2000000, 3000000})
    metric_connections = prometheus:gauge(
        "http_connections", "number of http connections", 
        {"state"})
end

local function log()
    local host = ngx.var.host
    local path = ngx.var.uri
    local status = ngx.var.status
    local user_agent = ngx.var.http_user_agent
    local country = ngx.var.geoip2_data_country_code
    local client = "pc"
    if string.match(user_agent, "Reader") then
        if string.match(user_agent, "[iI][oO][sS]") then
            client = "ios"
        else
            client = "android"
        end
    elseif string.match(user_agent, "iPhone|iPad|iPod|Android|Mobi") then
        client = "mobile"
    end

    local version = string.match(user_agent, "Reader.*/%d+%.%d+%.%d+")
    if version then
        version = string.gsub(version, ".*/", "")
    else
        version = ""
    end

    metric_requests:inc(1, {host, path, status, client, version, country})
    metric_request_latency:observe(tonumber(ngx.var.request_time), {host, path, country})
    metric_connection_rtt:observe(tonumber(ngx.var.tcpinfo_rtt), {country})
    metric_connection_rtt_var:observe(tonumber(ngx.var.tcpinfo_rttvar), {country})
end

local function display()
    metric_connections:set(ngx.var.connections_reading, {"reading"})
    metric_connections:set(ngx.var.connections_waiting, {"waiting"})
    metric_connections:set(ngx.var.connections_writing, {"writing"})
    prometheus:collect()
end


local _metamethod = {
        init = init,
        log = log,
        display = display
}

return _metamethod

```


