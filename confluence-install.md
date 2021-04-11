# Installing Confluence

### 系统要求
- CPU: 2C+
- RAM: 4G+
- HD: 50G+


### 下载 Confluence Server
https://www.atlassian.com/software/confluence/download


### 安装 Confluence
```
chmod +x atlassian-confluence-7.4.8-x64.bin
./atlassian-confluence-7.4.8-x64.bin
```

### 开放端口
```
firewall-cmd --permanent --add-port=8090/tcp
firewall-cmd --reload
```

/etc/init.d/confluence restart


