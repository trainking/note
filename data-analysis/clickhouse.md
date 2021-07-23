# ClickHouse

## 安装部署

### docker

```
docker pull yandex/clickhouse-server
docker run -d --name ck-server --ulimit nofile=262144:262144 -p 8123:8123 -p 9000:9000 -p 9009:9009 yandex/clickhouse-server
```