# FreeRADIUS Code Wiki

## 目录
- [项目概述](#项目概述)
- [整体架构](#整体架构)
- [主要模块职责](#主要模块职责)
- [认证系统详解](#认证系统详解)
- [授权系统详解](#授权系统详解)
- [计费系统详解](#计费系统详解)
- [关键类与函数](#关键类与函数)
- [依赖关系](#依赖关系)
- [配置文件详解](#配置文件详解)
- [项目运行方式](#项目运行方式)
- [适用场景与示例](#适用场景与示例)

---

## 项目概述

### 项目基本信息
- **项目名称**: FreeRADIUS Server
- **当前版本**: 4.0
- **主要功能**: 多协议策略服务器，支持RADIUS、DHCPv4、DHCPv6、DNS、TACACS+、VMPS
- **许可证**: GNU GPLv2
- **应用场景**: 网络接入认证、授权和计费（AAA）

### 项目简介
FreeRADIUS是一个高性能、高度可配置的多协议策略服务器。该项目是全球使用最广泛的开源RADIUS服务器，被全球数十亿用户的网络接入提供认证服务，从10用户规模到千万级用户规模都能支持。

---

## 整体架构

### 系统架构图
FreeRADIUS采用模块化、可扩展的架构设计，主要由以下层次化组件构成：

```
┌─────────────────────────────────────────────────────────┐
│                    网络接入层                       │
│  (RADIUS/DHCP/DNS/TACACS+/VMPS协议处理           │
└───────────────────┬─────────────────────────────────┘
                   │
┌───────────────────▼─────────────────────────────────┐
│                  监听器层                             │
│  (监听网络请求、解码协议数据包                     │
└───────────────────┬─────────────────────────────────┘
                   │
┌───────────────────▼─────────────────────────────────┐
│                 处理层                              │
│  (Virtual Server / unlang策略处理               │
└───────────────────┬─────────────────────────────────┘
                   │
┌───────────────────▼─────────────────────────────────┐
│                模块层                             │
│  (认证/授权/计费/数据库/外部接口模块         │
└───────────────────┬─────────────────────────────────┘
                   │
┌───────────────────▼─────────────────────────────────┐
│               基础库层                           │
│  (util/server/io/eap/tls/redis/ldap等        │
└─────────────────────────────────────────────────┘
```

### 核心目录结构
```
/workspace/
├── src/                  # 源代码目录
│   ├── bin/             # 可执行文件
│   ├── lib/             # 核心库
│   │   ├── server/      # 服务器核心库
│   │   ├── util/      # 工具库
│   │   ├── eap/       # EAP协议库
│   │   ├── tls/       # TLS库
│   │   ├── unlang/    # 策略语言实现
│   │   └── ...
│   ├── modules/        # 模块代码
│   │   └── rlm_*/   # 各类模块
│   ├── protocols/     # 协议实现
│   └── ...
├── raddb/              # 配置文件目录
│   ├── mods-available/ # 可用模块配置
│   ├── mods-enabled/     # 已启用模块配置
│   ├── sites-available/ # 可用站点配置
│   ├── policy.d/       # 策略配置
│   └── ...
├── doc/                # 文档
└── ...
```

---

## 主要模块职责

### 核心库
- **lib/server**: 服务器核心功能，包括配置解析、请求处理、模块加载等
- **lib/util**: 通用工具函数，包括字符串处理、数据结构、日志等
- **lib/eap**: EAP协议实现
- **lib/tls**: TLS/SSL加密支持
- **lib/unlang**: unlang策略语言解释器
- **lib/ldap**: LDAP客户端库
- **lib/redis**: Redis客户端库

### 模块分类
FreeRADIUS的模块分为以下几类：

#### 1. 认证模块 (Authentication)
- `rlm_pap`: PAP认证
- `rlm_chap`: CHAP认证
- `rlm_mschap`: MS-CHAP认证
- `rlm_eap`: EAP认证框架
- `rlm_krb5`: Kerberos认证
- `rlm_pam`: PAM认证
- `rlm_ldap`: LDAP认证
- `rlm_sql`: SQL数据库认证

#### 2. 授权模块 (Authorization)
- `rlm_files`: 基于文件的用户数据库
- `rlm_unix`: Unix系统用户
- `rlm_ldap`: LDAP目录查询
- `rlm_sql`: SQL数据库查询

#### 3. 计费模块 (Accounting)
- `rlm_detail`: 详细日志记录
- `rlm_sql`: SQL数据库计费记录
- `rlm_radutmp`: radutmp记录
- `rlm_linelog`: 行日志

#### 4. 数据库模块
- `rlm_sql`: SQL数据库支持
- `rlm_ldap`: LDAP目录支持
- `rlm_redis`: Redis缓存
- `rlm_kafka`: Kafka消息队列

---

## 认证系统详解

### 认证概述
认证是验证用户身份的过程，FreeRADIUS支持多种认证方式。

### 1. PAP (Password Authentication Protocol)

#### 原理
PAP是最基本的认证方式，用户密码以明文（经过RADIUS共享密钥加密）传输，服务器验证密码是否正确。

#### 相关配置
**文件位置: [raddb/mods-available/pap](file:///workspace/raddb/mods-available/pap)

配置示例:
```
pap {
    # 自动检测User-Password属性
    # normalize = yes
    # header = "%{User-Name}"
}
```

#### 处理流程
1. NAS发送Access-Request包，包含User-Name和User-Password属性
2. rlm_pap模块接收请求
3. 从请求中提取User-Password
4. 与存储的密码进行比较
5. 返回Access-Accept或Access-Reject

### 2. CHAP (Challenge-Handshake Authentication Protocol)

#### 原理
CHAP是挑战握手认证协议，使用三次握手机制，避免明文传输密码。

#### 相关配置
**文件位置**: [raddb/mods-available/chap](file:///workspace/raddb/mods-available/chap)

配置示例:
```
chap {
    # 自动检测CHAP-Challenge和CHAP-Password属性
}
```

#### 处理流程
1. NAS生成随机挑战值（CHAP-Challenge）
2. 用户使用MD5哈希(Challenge + Password生成CHAP-Password
3. 服务器重新计算哈希值进行验证

### 3. MS-CHAP (Microsoft Challenge Handshake Authentication Protocol)

#### 原理
MS-CHAP是微软扩展的CHAP协议，支持更强的安全性和额外功能。

#### 相关配置
**文件位置**: [raddb/mods-available/mschap](file:///workspace/raddb/mods-available/mschap)

配置示例:
```
mschap {
    use_mppe = yes
    require_encryption = yes
    require_strong = yes
}
```

#### 处理流程
1. 客户端使用NT密码哈希进行挑战响应
2. 支持MPPE加密支持
3. 提供NTLM认证集成

### 4. EAP (Extensible Authentication Protocol)

#### 原理
EAP是可扩展认证协议框架，支持多种认证方法。

#### 相关配置
**文件位置**: [raddb/mods-available/eap](file:///workspace/raddb/mods-available/eap)

配置示例:
```
eap {
    default_eap_type = peap
    timer_expire = 60
    ignore_unknown_eap_types = no
    
    tls-config tls-common {
        private_key_password = whatever
        certificate_file = ${certdir}/server.pem
        ca_file = ${certdir}/ca.pem
        dh_file = ${certdir}/dh
        random_file = /dev/urandom
    }
    
    tls {
        tls = tls-common
    }
    
    peap {
        tls = tls-common
        default_eap_type = mschapv2
    }
}
```

#### EAP方法
- **EAP-TLS: 基于证书的认证
- **EAP-TTLS: 隧道化TLS
- **PEAP: 受保护的EAP
- **EAP-SIM: SIM卡认证
- **EAP-AKA: AKA认证
- **EAP-MD5: MD5挑战

#### 处理流程
1. EAP身份协商阶段 → TLS隧道建立 → 内层认证

### 5. LDAP认证

#### 原理
通过LDAP目录服务器进行用户认证。

#### 相关配置
**文件位置**: [raddb/mods-available/ldap](file:///workspace/raddb/mods-available/ldap)

配置示例:
```
ldap {
    server = 'ldap.example.com'
    identity = 'cn=admin,dc=example,dc=com'
    password = 'secret'
    base_dn = 'ou=people,dc=example,dc=com'
    user_filter = '(uid=%{%{Stripped-User-Name}:-%{User-Name}})'
}
```

### 6. SQL认证

#### 原理
通过SQL数据库进行用户认证。

#### 相关配置
**文件位置**: [raddb/mods-available/sql](file:///workspace/raddb/mods-available/sql)

配置示例:
```
sql {
    driver = 'rlm_sql_mysql'
    server = 'localhost'
    port = 3306
    login = 'radius'
    password = 'radpass'
    db = 'radius'
    
    authorize_check_query = "SELECT id, username, attribute, value, op \
                         FROM ${authcheck_table} \
                         WHERE username = '%{Stripped-User-Name:-%{User-Name}}' \
                         ORDER BY id"
}
```

---

## 授权系统详解

### 授权概述
授权是确定用户可以访问哪些资源和服务的过程。

### 主要授权机制

#### 1. 文件授权 (rlm_files)

**配置文件: [raddb/mods-available/files](file:///workspace/raddb/mods-available/files)

配置示例:
```
files {
    usersfile = ${confdir}/users
    compat = no
}
```

**users文件格式**:
```
user1  Cleartext-Password := "password1"
       Service-Type = Framed-User,
       Framed-Protocol = PPP,
       Framed-IP-Address = 192.168.1.10
```

#### 2. SQL授权

配置示例:
```
authorize_reply_query = "SELECT id, username, attribute, value, op \
                     FROM ${authreply_table} \
                     WHERE username = '%{Stripped-User-Name:-%{User-Name}}' \
                     ORDER BY id"
```

#### 3. LDAP授权

配置示例:
```
ldap {
    groupname_attribute = cn
    groupmembership_filter = "(|(&(objectClass=GroupOfNames)(objectClass=GroupOfUniqueNames))"
    groupmembership_attribute = member
}
```

### 授权处理流程
1. 接收认证成功后进入授权阶段
2. 查询用户属性和权限信息
3. 应用访问控制策略
4. 返回授权属性

---

## 计费系统详解

### 计费概述
计费是记录用户网络使用情况的过程，包括开始时间、结束时间、流量等信息。

### 主要计费方式

#### 1. 详细日志计费 (rlm_detail)

**配置文件**: [raddb/mods-available/detail](file:///workspace/raddb/mods-available/detail)

配置示例:
```
detail {
    filename = ${radacctdir}/%{Client-IP-Address}/detail-%Y%m%d
    permissions = 0600
    header = "%t"
}
```

#### 2. SQL计费

配置示例:
```
sql {
    accounting_start_query = "INSERT INTO ${acct_table} \
                          (acctsessionid, acctuniqueid, username, realm, \
                           nasipaddress, nasportid, nasporttype, \
                           acctstarttime, acctstoptime, acctsessiontime, \
                           acctauthentic, connectinfo_start, connectinfo_stop, \
                           acctinputoctets, acctoutputoctets, calledstationid, \
                           callingstationid, acctterminatecause, servicetype, \
                           framedprotocol, framedipaddress) \
                          VALUES \
                          ('%{Acct-Session-Id}', '%{Acct-Unique-Session-Id}', \
                           '%{SQL-User-Name}', '%{SQL-Realm}', \
                           '%{NAS-IP-Address}', '%{NAS-Port-Id}', '%{NAS-Port-Type}', \
                           FROM_UNIXTIME(%{integer:Event-Timestamp}), NULL, 0, \
                           '%{Acct-Authentic}', '%{Connect-Info}', '', \
                           0, 0, '%{Called-Station-Id}', '%{Calling-Station-Id}', '', \
                           '%{Service-Type}', '%{Framed-Protocol}', '%{Framed-IP-Address}')"
}
```

#### 3. radutmp计费

**配置文件**: [raddb/mods-available/radutmp](file:///workspace/raddb/mods-available/radutmp)

配置示例:
```
radutmp {
    filename = ${radacctdir}/radutmp
    permissions = 0644
    username = "%{User-Name}"
    caller_id = "%{Calling-Station-Id}"
}
```

### 计费类型
- **Start**: 会话开始
- **Interim-Update**: 会话中间更新
- **Stop**: 会话结束
- **On/Off**: NAS启动/关闭
- **Accounting-Off**: 整个NAS停止

### 计费流程
1. 接收计费请求 → 记录计费信息 → 更新会话状态 → 返回计费响应

---

## 关键类与函数

### 核心数据结构

#### REQUEST 结构
表示一个RADIUS请求，包含请求包、响应包、会话状态等信息。

#### MODULE 结构
表示一个加载的模块，包含模块名称、配置、实例等信息。

#### fr_server_t 结构
服务器实例结构。

### 主要函数

#### 请求处理
- `fr_request_alloc()
- `fr_request_process()`
- `fr_request_free()`

#### 模块操作
- `fr_module_load()`
- `fr_module_instantiate()`
- `fr_module_detach()`

#### 配置解析
- `cf_section_parse()`
- `cf_section_name2()`
- `cf_pair_find()`

#### 属性操作
- `fr_pair_add()`
- `fr_pair_find()`
- `fr_pair_delete()`

---

## 依赖关系

### 核心依赖
- **OpenSSL**: TLS/SSL加密
- **talloc**: 内存分配器
- **libkqueue/libev**: 事件循环（可选）

### 可选依赖
- **MySQL/PostgreSQL/Oracle**: SQL数据库
- **OpenLDAP**: LDAP支持
- **libkrb5**: Kerberos认证
- **libcurl**: HTTP客户端
- **hiredis**: Redis客户端
- **librdkafka**: Kafka客户端

---

## 配置文件详解

### 主配置文件: radiusd.conf

**文件位置**: [raddb/radiusd.conf.in](file:///workspace/raddb/radiusd.conf.in)

主要配置项:
```
# 服务器名称
name = radiusd

# 日志配置
log {
    destination = file
    file = ${logdir}/radius.log
    colourise = yes
}

# 线程池配置
thread pool {
    num_workers = 1
}

# 安全配置
security {
    max_attributes = 200
}
```

### 客户端配置: clients.conf

**文件位置**: [raddb/clients.conf](file:///workspace/raddb/clients.conf)

配置示例:
```
client localhost {
    ipaddr = 127.0.0.1
    secret = testing123
    require_message_authenticator = auto
}
```

### 虚拟服务器配置

**目录**: [raddb/sites-available/](file:///workspace/raddb/sites-available/)

主要处理阶段:
- `recv Access-Request`: 接收认证请求
- `authorize`: 授权阶段
- `authenticate`: 认证阶段
- `post-auth`: 认证后处理
- `recv Accounting-Request`: 接收计费请求
- `preacct`: 计费前处理
- `accounting`: 计费处理

---

## 项目运行方式

### 编译安装

```bash
# 配置
./configure --prefix=/usr/local/freeradius

# 编译
make

# 安装
make install
```

### 运行服务器

```bash
# 调试模式（推荐，前台运行，输出详细日志
radiusd -X

# 后台运行
radiusd

# 指定配置文件
radiusd -d /path/to/config

# 指定名称
radiusd -n myserver
```

### 测试工具

```bash
# radtest: 简单测试认证
radtest user password localhost 0 testing123

# radclient: 通用RADIUS客户端
echo "User-Name=user,User-Password=password" | radclient -x localhost auth testing123

# raddebug: 调试指定用户
raddebug -u username
```

---

## 适用场景与示例

### 场景1: WiFi 802.1X认证

适用设备:
- 无线路由器/AP
- 企业WiFi网络

配置步骤:
1. 配置EAP模块（PEAP/MS-CHAPv2
2. 配置TLS证书
3. 配置用户数据库
4. 配置AP使用RADIUS认证

### 场景2: VPN认证

适用设备:
- VPN服务器
- 远程访问服务器

### 场景3: ISP拨号认证

适用场景:
- PPPoE服务器
- DSLAM设备

### 场景4: 企业网络接入控制

配置示例 (raddb/sites-available/default:
```
server default {
    listen {
        type = auth
        ipaddr = *
        port = 1812
    }
    
    listen {
        type = acct
        ipaddr = *
        port = 1813
    }
    
    authorize {
        filter_username
        preprocess
        suffix
        eap {
            ok = return
        }
        files
        sql
        ldap
        expiration
        logintime
        pap
    }
    
    authenticate {
        Auth-Type PAP {
            pap
        }
        Auth-Type CHAP {
            chap
        }
        Auth-Type MS-CHAP {
            mschap
        }
        eap
    }
    
    preacct {
        preprocess
        acct_unique
        suffix
        files
    }
    
    accounting {
        detail
        radutmp
        sradutmp
        sql
    }
    
    post-auth {
        Post-Auth-Type REJECT {
            attr_filter.access_reject
        }
    }
}
```

---

## 总结

FreeRADIUS是一个功能强大、高度可配置的AAA服务器，适用于各种网络认证、授权和计费场景。通过模块化设计和灵活的配置，能够满足从简单到复杂的各种需求。

本Code Wiki提供了项目的整体架构、主要模块、认证授权计费功能的详细说明，以及配置和运行指南。
