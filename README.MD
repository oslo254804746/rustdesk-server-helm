[中文文档](https://github.com/oslo254804746/rustdesk-server-helm/blob/master/README_zh.MD)


[English](https://github.com/oslo254804746/rustdesk-server-helm/blob/master/README.MD)

## Introduction
This is a chart for rustdesk-server

## Configuration Options
### Port Configuration
According to the official documentation of `rustdesk`, there are five ports in total, `21118-21119` are web ports, which are not required to be opened, 

and `21116` needs to include TCP/UDP. The default `PortType` is `ClusterIP`, which can be modified to `NodePort` or Proxy by yourself through  like `traefik crd`
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

### Volume Configuration
The default is EmptyDir, which is shared by two containers, storing pubKey and SQLite data
```yaml
volumes:
  - name: rustdesk-data
    emptyDir: {}
```
You can modify it yourself, such as changing it to hostPath
```yaml
volumes:
  - name: rustdesk-data
    hostPath:
      path: /mnt/rustdesk/
```


## Installation
`cd` to the directory
```shell
helm install rustdesk .
```
