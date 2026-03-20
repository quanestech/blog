# OpenLDAP + MySQL 完整部署指南

## 环境概述

- 操作系统: Linux (Ubuntu)
- 部署方式: Docker容器
- OpenLDAP: osixia/openldap
- MySQL: mysql:latest
- 域名: cryptox.io

---

## 一、部署 OpenLDAP

### 1.1 拉取镜像

```bash
docker pull osixia/openldap:latest
```

### 1.2 启动容器

```bash
docker run -d --name openldap \
  --hostname ldap.cryptox.io \
  -p 389:389 -p 636:636 \
  -e LDAP_ORGANISATION="CryptoX" \
  -e LDAP_DOMAIN="cryptox.io" \
  -e LDAP_ADMIN_PASSWORD="admin" \
  osixia/openldap:latest
```

### 1.3 验证安装

```bash
docker exec openldap ldapsearch -x -H ldap://localhost -b "dc=cryptox,dc=io" \
  -D "cn=admin,dc=cryptox,dc=io" -w admin
```

---

## 二、创建管理员账号

### 2.1 生成密码哈希

```bash
docker exec openldap slappasswd -s YourPassword
```

### 2.2 创建用户 LDIF 文件

```ldif
dn: cn=quan,dc=cryptox,dc=io
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: quan
sn: Admin
givenName: Quan
displayName: Quan Admin
uid: quan
uidNumber: 1000
gidNumber: 1000
userPassword: {SSHA}你的哈希值
homeDirectory: /home/quan
loginShell: /bin/bash
```

### 2.3 添加用户

```bash
docker exec openldap ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=cryptox,dc=io" -w admin -f /tmp/quan.ldif
```

---

## 三、创建用户组

### 3.1 管理员组

```ldif
dn: cn=admins,dc=cryptox,dc=io
objectClass: groupOfNames
cn: admins
member: cn=quan,dc=cryptox,dc=io
```

```bash
docker exec openldap ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=cryptox,dc=io" -w admin -f admins.ldif
```

### 3.2 普通用户组

```ldif
dn: cn=users,dc=cryptox,dc=io
objectClass: groupOfNames
cn: users
member: cn=quan,dc=cryptox,dc=io
```

```bash
docker exec openldap ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=cryptox,dc=io" -w admin -f users.ldif
```

---

## 四、部署 MySQL

### 4.1 拉取镜像

```bash
docker pull mysql:latest
```

### 4.2 启动容器

```bash
docker run -d --name mysql-ldap \
  -e MYSQL_ROOT_PASSWORD=root123 \
  -e MYSQL_DATABASE=ldap \
  -e MYSQL_USER=ldap \
  -e MYSQL_PASSWORD=ldap123 \
  -p 3306:3306 \
  mysql:latest
```

### 4.3 验证

```bash
docker exec mysql-ldap mysql -uroot -proot123 -e "SHOW DATABASES;"
```

---

## 五、数据备份到 MySQL

### 5.1 创建备份数据库

```sql
CREATE DATABASE IF NOT EXISTS ldap_backup;
USE ldap_backup;

CREATE TABLE ldap_entries (
    id INT AUTO_INCREMENT PRIMARY KEY,
    dn VARCHAR(512) NOT NULL,
    objectClass VARCHAR(256),
    cn VARCHAR(256),
    sn VARCHAR(256),
    uid VARCHAR(256),
    userPassword VARCHAR(256),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY idx_dn (dn)
);
```

### 5.2 导出 LDAP 数据

```bash
docker exec openldap ldapsearch -x -H ldap://localhost -b "dc=cryptox,dc=io" \
  -D "cn=admin,dc=cryptox,dc=io" -w admin > /tmp/ldap-backup.ldif
```

---

## 六、常用 LDAP 命令

### 查看所有条目
```bash
ldapsearch -x -H ldap://localhost -b "dc=cryptox,dc=io" \
  -D "cn=admin,dc=cryptox,dc=io" -w admin
```

### 搜索特定用户
```bash
ldapsearch -x -H ldap://localhost -b "dc=cryptox,dc=io" \
  -D "cn=admin,dc=cryptox,dc=io" -w admin "(cn=quan)"
```

### 添加新用户
```bash
ldapadd -x -H ldap://localhost -D "cn=admin,dc=cryptox,dc=io" \
  -w admin -f newuser.ldif
```

### 删除用户
```bash
ldapdelete -x -H ldap://localhost -D "cn=admin,dc=cryptox,dc=io" \
  -w admin "cn=username,dc=cryptox,dc=io"
```

### 修改密码
```bash
ldappasswd -x -H ldap://localhost -D "cn=admin,dc=cryptox,dc=io" \
  -w admin -s newpassword "cn=quan,dc=cryptox,dc=io"
```

---

## 七、信息汇总

| 项目 | 值 |
|------|-----|
| 基础 DN | dc=cryptox,dc=io |
| 组织名 | CryptoX |
| 管理员 DN | cn=admin,dc=cryptox,dc=io |
| 管理员密码 | admin |
| 用户 DN | cn=quan,dc=cryptox,dc=io |
| LDAP 端口 | 389 |
| MySQL 端口 | 3306 |

---

## 八、新增用户指南

### 8.1 创建新用户

1. 创建用户 LDIF 文件：

```ldif
dn: cn=newuser,dc=cryptox,dc=io
objectClass: inetOrgPerson
objectClass: posixAccount
cn: newuser
sn: User
givenName: New
uid: newuser
uidNumber: 1001
gidNumber: 100
userPassword: {SSHA}生成的哈希值
homeDirectory: /home/newuser
loginShell: /bin/bash
```

2. 添加用户：

```bash
docker exec openldap ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=cryptox,dc=io" -w admin -f newuser.ldif
```

### 8.2 将用户添加到组

```ldif
dn: cn=users,dc=cryptox,dc=io
changetype: modify
add: member
member: cn=newuser,dc=cryptox,dc=io
```

```bash
docker exec openldap ldapmodify -x -H ldap://localhost \
  -D "cn=admin,dc=cryptox,dc=io" -w admin -f add-to-group.ldif
```

### 8.3 创建新组

```ldif
dn: cn=developers,dc=cryptox,dc=io
objectClass: groupOfNames
cn: developers
member: cn=quan,dc=cryptox,dc=io
```

```bash
docker exec openldap ldapadd -x -H ldap://localhost \
  -D "cn=admin,dc=cryptox,dc=io" -w admin -f developers.ldif
```

---

*本文档详细记录了 OpenLDAP 和 MySQL 的完整部署过程。*
