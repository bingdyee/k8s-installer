# 安装MySQL

1. 创建mysql用户
```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

2. 解压到/usr/local/mysql
```
tar zxf mysql-8.0.20-el7-x86_64.tar.gz
mv mysql-8.0.20-el7-x86_64 /usr/local/mysql
```

3. 目录权限
```
chown -R mysql.mysql /usr/local/mysql
```

4. 卸载 mariadb-libs
```
yum remove mariadb-libs
```

5. 启用 mysql.service
```
cp support-files/mysql.server /etc/init.d/mysql
```


6. 启动 mysql
```
service mysql start
```

7. 设置密码
```
bin/mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
```

8. 开放防火墙
```
firewall-cmd --permanent --add-port=3306/tcp
firewall-cmd --reload
```

9. 基本操作验证
```
-- 创建库和用户
CREATE DATABASE confluence CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER confluence IDENTIFIED BY '123456';
GRANT all ON confluence.* TO 'confluence'@'%';
FLUSH PRIVILEGES;

-- 撤销用户权限
REVOKE all ON phorcys_oauth.* FROM 'devuser'@'%';
REVOKE all ON *.* FROM 'devuser'@'%';

-- 查看用户权限
SHOW GRANTS FOR 'devuser'@'%';

-- 删除用户
DROP USER 'devuser'@'%';

-- 刷新系统权限
FLUSH PRIVILEGES;
```