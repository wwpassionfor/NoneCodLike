# Nginx安全部署与端口加固报告

> **执行日期**：2026-05-20
> **目标**：配置Nginx反向代理+安全头+端口隔离，确保仅80/443/22对外
> **状态**：✅ 已完成（HTTP部分），⏳ HTTPS待域名注册后配置

---

## 一、部署架构

### 1.1 网络拓扑

```
                        阿里云安全组
                        80/443/22
                            │
                    ┌───────┴───────┐
                    │  Docker Nginx │
                    │   :80 (对外)   │
                    └───────┬───────┘
                            │ Docker Bridge (yuyuan-network)
            ┌───────┬───────┼───────┬───────┐
            │       │       │       │       │
        Backend  Grafana  Prometheus MySQL  Redis
       :9199    :3000    :9090    :3306   :6379
      (内部)   (内部)   (内部)   (内部)  (内部)
```

### 1.2 访问路径

| 外部URL | 内部服务 | 代理方式 |
|---------|----------|----------|
| http://your-server-ip/ | 静态文件 /usr/share/nginx/portfolio | 直接服务 |
| http://your-server-ip/api/ | thumb-backend:9199 | 反向代理+路径重写 |
| http://your-server-ip/doc.html | thumb-backend:9199/api/doc.html | 反向代理 |
| http://your-server-ip/grafana/ | thumb-grafana:3000 | 反向代理 |
| http://your-server-ip/prometheus/ | thumb-prometheus:9090 | 反向代理 |

---

## 二、Nginx配置详情

### 2.1 完整配置文件

文件位置：`deploy/nginx/default.conf`

```nginx
upstream backend {
    server thumb-backend:9199;
}

upstream grafana {
    server thumb-grafana:3000;
}

upstream prometheus {
    server thumb-prometheus:9090;
}

server {
    listen 80;
    server_name nonecodlike.fun www.nonecodlike.fun your-server-ip;

    client_max_body_size 10m;

    # 安全头
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # SPA路由
    location / {
        root /usr/share/nginx/portfolio;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # API反向代理
    location /api/ {
        proxy_pass http://backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    # Knife4j API文档
    location /doc.html {
        proxy_pass http://backend/api/doc.html;
    }

    location /v3/api-docs {
        proxy_pass http://backend/api/v3/api-docs;
    }

    location /swagger-resources {
        proxy_pass http://backend/api/swagger-resources;
    }

    location /webjars/ {
        proxy_pass http://backend/api/webjars/;
    }

    # Grafana监控面板
    location /grafana/ {
        proxy_pass http://grafana/;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Prometheus指标
    location /prometheus/ {
        proxy_pass http://prometheus/;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        root /usr/share/nginx/portfolio;
        expires 30d;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
}
```

### 2.2 HTTPS配置（待域名后启用）

文件位置：`deploy/nginx/portfolio-https.conf`

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

server {
    listen 443 ssl http2;
    server_name nonecodlike.fun;

    ssl_certificate /etc/letsencrypt/live/nonecodlike.fun/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/nonecodlike.fun/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    # ... 其余同HTTP配置
}

server {
    listen 80;
    server_name nonecodlike.fun;
    return 301 https://$server_name$request_uri;
}
```

---

## 三、端口安全加固

### 3.1 Docker Compose端口绑定变更

**变更前**（所有端口对外暴露）：

```yaml
ports:
  - "9199:9199"    # 0.0.0.0:9199 → 外部可访问
  - "3306:3306"    # 0.0.0.0:3306 → 外部可访问
  - "6379:6379"    # 0.0.0.0:6379 → 外部可访问
```

**变更后**（仅本地绑定）：

```yaml
ports:
  - "127.0.0.1:9199:9199"    # 仅本地访问
  - "127.0.0.1:3306:3306"    # 仅本地访问
  - "127.0.0.1:6379:6379"    # 仅本地访问
```

### 3.2 加固验证结果

```bash
$ ss -tlnp | grep docker-proxy
LISTEN 0 4096 127.0.0.1:9199  ✅ 仅本地
LISTEN 0 4096 0.0.0.0:80      ✅ 对外（Nginx）
LISTEN 0 4096 127.0.0.1:3000  ✅ 仅本地
LISTEN 0 4096 127.0.0.1:9090  ✅ 仅本地
LISTEN 0 4096 127.0.0.1:3306  ✅ 仅本地
LISTEN 0 4096 127.0.0.1:6379  ✅ 仅本地
LISTEN 0 4096 127.0.0.1:8080  ✅ 仅本地
LISTEN 0 4096 127.0.0.1:6650  ✅ 仅本地
```

### 3.3 阿里云安全组配置指引

**目标**：仅开放 80/443/22 端口

操作步骤：
1. 登录 [阿里云控制台](https://swasnext.console.aliyun.com/servers/cn-hangzhou)
2. 选择实例 → 防火墙/安全组
3. 配置规则：

| 协议 | 端口 | 源地址 | 说明 |
|------|------|--------|------|
| TCP | 22 | 0.0.0.0/0 | SSH远程管理 |
| TCP | 80 | 0.0.0.0/0 | HTTP访问 |
| TCP | 443 | 0.0.0.0/0 | HTTPS访问 |
| TCP | 9199 | ❌ 删除 | 后端API（通过Nginx代理） |
| TCP | 3000 | ❌ 删除 | Grafana（通过Nginx代理） |
| TCP | 9090 | ❌ 删除 | Prometheus（通过Nginx代理） |
| TCP | 3306 | ❌ 删除 | MySQL（仅内部） |
| TCP | 6379 | ❌ 删除 | Redis（仅内部） |

---

## 四、安全头验证

### 4.1 测试命令与结果

```bash
$ curl -sI http://your-server-ip/ | grep -E 'X-Frame|X-Content|X-XSS|Referrer'
X-Frame-Options: DENY ✅
X-Content-Type-Options: nosniff ✅
X-XSS-Protection: 1; mode=block ✅
Referrer-Policy: strict-origin-when-cross-origin ✅
```

### 4.2 安全头说明

| 安全头 | 值 | 防护目标 |
|--------|------|----------|
| X-Frame-Options | DENY | 防止点击劫持（Clickjacking） |
| X-Content-Type-Options | nosniff | 防止MIME类型嗅探 |
| X-XSS-Protection | 1; mode=block | 浏览器XSS过滤 |
| Referrer-Policy | strict-origin-when-cross-origin | 控制Referer泄露 |
| HSTS | ⏳ 域名后启用 | 强制HTTPS |
| CSP | ⏳ 域名后启用 | 内容安全策略 |

---

## 五、问题与解决

### 5.1 Nginx启动失败（80端口占用）

**问题**：`nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)`

**原因**：Docker thumb-nginx容器已占用80端口

**解决**：不使用系统Nginx，将作品集配置整合到Docker Nginx容器中

### 5.2 Knife4j 404

**问题**：`http://localhost/doc.html` 返回404

**原因**：Spring Boot context-path为`/api`，Knife4j实际路径为`/api/doc.html`

**解决**：Nginx代理路径修正为`proxy_pass http://backend/api/doc.html`

### 5.3 Prometheus容器启动失败

**问题**：`error mounting ... prometheus.yml: not a directory`

**原因**：之前scp上传时prometheus.yml被创建为目录

**解决**：`rm -rf /root/yu-like-main/deploy/prometheus.yml`，重新上传文件

### 5.4 后端服务502

**问题**：Nginx重启后Doc/Grafana返回502

**原因**：后端Java应用启动需要60-90秒，Nginx先于后端就绪

**解决**：等待后端健康检查通过后自动恢复

---

## 六、SSL证书配置计划

### 6.1 前置条件

- [x] Nginx运行正常
- [x] 域名DNS解析到服务器
- [x] 域名注册完成 (nonecodlike.fun)
- [ ] DNS A记录配置

### 6.2 Let's Encrypt证书获取

```bash
# 安装certbot
apt-get install -y certbot

# 获取证书（standalone模式，需先停止Nginx）
docker stop thumb-nginx
certbot certonly --standalone -d nonecodlike.fun -d www.nonecodlike.fun

# 证书路径
# /etc/letsencrypt/live/nonecodlike.fun/fullchain.pem
# /etc/letsencrypt/live/nonecodlike.fun/privkey.pem

# 自动续期
echo "0 3 * * * certbot renew --quiet && docker restart thumb-nginx" | crontab -
```

### 6.3 HTTPS配置步骤

1. 将`portfolio-https.conf`复制到`/etc/nginx/conf.d/`
2. 挂载证书文件到Docker Nginx容器
3. 更新docker-compose.yml添加443端口和证书卷
4. 重启Nginx容器
