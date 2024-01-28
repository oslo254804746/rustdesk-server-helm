[中文文档](https://github.com/oslo254804746/rustdesk-server-helm/blob/master/README_zh.MD)


[English](https://github.com/oslo254804746/rustdesk-server-helm/blob/master/README.MD)


## 介绍
这是一个`rustdesk-server`的`chart`

## 配置项

### 端口配置

根据`rustdesk`官方文档, 一共会有5个端口,其中`21118-21119`为`web`端口, 非必须开启, `21116` 需包含`TCP/UDP`, 默认是`ClusterIP`, 可以修改为`NodePort`或者
通过`traefik crd` 自行代理

```yaml
portConfigs:
  hbbr:
    type: ClusterIP
    replayPort:
      port: 21117
      targetPort: 21117
    clientPort:
      enable: false
      port: 21119
      targetPort: 21119
  hbbs:
      type: ClusterIP
      natPort:
        port: 21115
        protocol: TCP
        targetPort: 21115
      registryPort:
        port: 21116
        targetPort: 21116
      heartbeatPort:
        port: 21116
        targetPort: 21116
      webPort:
        port: 21118
        targetPort: 21118
        enable: false
        protocol: TCP

```

### 卷配置
默认是`EmptyDir`, 两个容器共用，存放`pubKey 和 SQLite数据`
```yaml
volumes:
  - name: rustdesk-data
    emptyDir: {}
```
可自行修改,如修改为`hostPath`
```yaml
volumes:
  - name: rustdesk-data
    hostPath:
      path: /mnt/rustdesk/
```

## 安装

`cd` 到目录

```shell
helm install rustdesk .
```
