
```
# HTTP 代理
export http_proxy=http://10.15.136.20:8080

# HTTPS 代理
export https_proxy=http://10.15.136.20:8080

# 如果需要认证
export http_proxy=http://username:password@10.15.136.20:8080

# 不加代理的地址（内网、localhost等）
export no_proxy=localhost,127.0.0.1,.local,.internal,10.0.0.0/8
```