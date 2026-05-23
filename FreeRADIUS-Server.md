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

主要配置项说明:

#### 基本配置
```
# 服务器名称
name = radiusd

# 路径配置
prefix = /usr
sysconfdir = ${prefix}/etc
localstatedir = ${prefix}/var
logdir = ${localstatedir}/log/radius
confdir = ${sysconfdir}/raddb
radacctdir = ${localstatedir}/log/radius/radacct
libdir = ${prefix}/lib
```

#### 请求处理配置
```
request {
    # 最大请求数（每个线程）
    max = 16384
    
    # 请求超时时间（秒）
    timeout = 30
    
    # 请求重用配置
    reuse {
        min = 10
        # max = 100
        cleanup_interval = 30s
    }
}
```

#### 日志配置
```
log {
    # 日志目标：file/syslog/stdout/stderr
    destination = file
    
    # 是否彩色输出
    colourise = yes
    
    # 日志文件路径
    file = ${logdir}/radius.log
    
    # syslog配置
    syslog_facility = daemon
    
    # 是否隐藏敏感信息
    suppress_secrets = yes
}
```

#### 线程池配置
```
thread pool {
    # 网络线程数
    # num_networks = 1
    
    # 工作线程数
    # num_workers = 1
    
    # OpenSSL异步上下文池配置
    # openssl_async_pool_init = 64
    # openssl_async_pool_max = 1024
}
```

#### 安全配置
```
security {
    # 运行服务器的用户和组
    # user = radius
    # group = radius
    
    # 是否允许核心转储
    allow_core_dumps = no
    
    # 最大属性数
    max_attributes = 200
}
```

### 客户端配置: clients.conf

**文件位置**: [raddb/clients.conf](file:///workspace/raddb/clients.conf)

客户端配置详细说明:

```
client localhost {
    # 客户端IP地址或网络范围（可使用CIDR格式）
    ipaddr = 127.0.0.1
    # ipv4addr = *
    # ipv6addr = ::
    
    # 传输协议（udp/tcp/*）
    proto = *
    
    # 共享密钥（必须修改！）
    secret = testing123
    
    # 是否要求Message-Authenticator（yes/no/auto）
    require_message_authenticator = auto
    
    # Proxy-State限制（yes/no/auto）
    limit_proxy_state = auto
    
    # TCP连接限制
    limit {
        # 最大连接数
        max_connections = 16
        
        # 连接生命周期（秒），0表示永久
        lifetime = 0
        
        # 空闲超时（秒）
        idle_timeout = 30
    }
    
    # 短名称（可选）
    # shortname = localhost
}
```

**客户端配置参数说明**:
- `ipaddr/ipv4addr/ipv6addr`: 客户端地址，支持单个IP或CIDR网络范围
- `secret`: 与NAS设备共享的密钥，用于加密和签名RADIUS数据包
- `require_message_authenticator`: 强制要求Access-Request包含Message-Authenticator属性，防止爆破攻击
- `limit_proxy_state`: 限制Proxy-State属性以防止某些攻击

### 模块配置详解

#### PAP模块配置

**文件位置**: [raddb/mods-available/pap](file:///workspace/raddb/mods-available/pap)

```
pap {
    # 是否自动识别base64或hex编码的密码
    # normalise = no
    
    # 使用哪个属性作为用户密码
    # password_attribute = User-Password
}
```

**PAP支持的密码格式**:
- `Password.Cleartext`: 明文密码
- `Password.Crypt`: Unix crypt加密密码
- `Password.MD5`: MD5哈希密码
- `Password.SMD5`: 带盐的MD5哈希
- `Password.SHA1`: SHA1哈希密码
- `Password.SSHA`: 带盐的SHA1哈希
- `Password.SHA2/SHA224/SHA256/SHA384/SHA512`: SHA2系列哈希
- `Password.SSHA2-224/SSHA2-256/SSHA2-384/SSHA2-512`: 带盐的SHA2系列哈希
- `Password.NT/MD4`: Windows NT哈希
- `Password.LM`: Windows LM哈希

#### CHAP模块配置

**文件位置**: [raddb/mods-available/chap](file:///workspace/raddb/mods-available/chap)

CHAP模块不需要配置，会自动处理CHAP-Challenge和CHAP-Password属性。

**注意**: CHAP认证需要访问用户的明文密码。

#### Files模块配置

**文件位置**: [raddb/mods-available/files](file:///workspace/raddb/mods-available/files)

```
files {
    # 模块配置目录
    moddir = ${modconfdir}/${.:instance}
    
    # 用于匹配的键属性
    # key = "%{Stripped-User-Name || User-Name}"
    
    # 用户文件路径
    filename = ${moddir}/authorize
    
    # 匹配后设置的属性
    # match_attr = control.User-Category
    
    # v3兼容性模式
    # v3_compat = no
}
```

**users文件格式说明**:
```
用户名  检查项...
        回复项...

示例:
user1  Cleartext-Password := "password1"
       Service-Type = Framed-User,
       Framed-Protocol = PPP,
       Framed-IP-Address = 192.168.1.10
```

#### SQL模块配置

**文件位置**: [raddb/mods-available/sql](file:///workspace/raddb/mods-available/sql)

```
sql {
    # SQL方言：cassandra/firebird/mysql/mssql/oracle/postgresql/sqlite
    dialect = "sqlite"
    
    # 驱动模块，通常与方言相同
    driver = "${dialect}"
    
    # 包含驱动特定配置
    $INCLUDE ${modconfdir}/sql/driver/${driver}
    
    # 数据库连接信息
    # server = "localhost"
    # port = 3306
    # login = "radius"
    # password = "radpass"
    
    # 数据库名称
    radius_db = "radius"
    
    # Oracle连接字符串格式
    # radius_db = "(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))(CONNECT_DATA=(SID=your_sid)))"
    
    # PostgreSQL连接字符串格式
    # radius_db = "dbname=radius host=localhost user=radius password=radpass"
    
    # 计费表配置
    acct_table1 = "radacct"
    acct_table2 = "radacct"
    
    # 认证后记录表
    postauth_table = "radpostauth"
    
    # 检查项表
    authcheck_table = "radcheck"
    groupcheck_table = "radgroupcheck"
    
    # 回复项表
    authreply_table = "radreply"
    groupreply_table = "radgroupreply"
    
    # 用户组表
    usergroup_table = "radusergroup"
    
    # 是否读取组信息
    # read_groups = yes
    
    # 是否读取用户配置文件
    # read_profiles = yes
    
    # SQL查询日志文件
    # logfile = ${logdir}/sqllog.sql
    
    # 查询超时（Cassandra和unixodbc）
    # query_timeout = 5
    
    # 连接池配置
    pool {
        # 初始化连接数
        start = 0
        
        # 最小连接数
        min = 1
        
        # 最大连接数
        max = 100
        
        # 同时连接数
        connecting = 2
        
        # 连接使用次数限制（0表示无限）
        uses = 0
        
        # 连接生命周期（秒）
        lifetime = 0
        
        # 打开延迟（秒）
        # open_delay = 0.2
        
        # 关闭延迟（秒）
        # close_delay = 10
        
        # 管理间隔（秒）
        # manage_interval = 0.2
    }
    
    # 组属性名称
    group_attribute = "${.:instance}-Group"
    
    # 是否缓存组信息
    # cache_groups = no
    
    # 记录成功查询编号的属性
    # query_number_attribute = 'Query-Number'
    
    # 包含数据库特定查询
    $INCLUDE ${modconfdir}/${.:name}/main/${dialect}/queries.conf
}
```

**SQL数据库表结构**:
- `radcheck`: 用户检查项
- `radreply`: 用户回复项
- `radgroupcheck`: 用户组检查项
- `radgroupreply`: 用户组回复项
- `radusergroup`: 用户-组映射
- `radacct`: 计费记录
- `radpostauth`: 认证后记录

#### EAP模块配置

**文件位置**: [raddb/mods-available/eap](file:///workspace/raddb/mods-available/eap)

```
eap {
    # 是否要求EAP身份包含realm（nai/yes/no）
    # require_identity_realm = nai
    
    # 默认EAP类型
    # default_eap_type = md5
    
    # 是否忽略未知的EAP类型
    ignore_unknown_eap_types = no
    
    # 允许的EAP类型列表
    type = md5
    # type = pwd
    type = gtc
    type = tls
    type = ttls
    type = mschapv2
    type = peap
    # type = fast
    # type = aka
    # type = sim
    
    # EAP-MD5配置（不推荐用于无线）
    md5 {
    }
    
    # EAP-PWD配置
    pwd {
        # 椭圆曲线组
        group = 19
        
        # 服务器标识
        server_id = theserver@example.com
        
        # 分片大小
        fragment_size = 1020
    }
    
    # EAP-GTC配置
    gtc {
        # 挑战文本
        challenge = "Password: "
        
        # 认证模式（PAP/Local/Accept）
        auth_type = PAP
    }
    
    # TLS配置（被tls/ttls/peap/fast共享）
    tls-config tls-common {
        # 私钥密码
        private_key_password = whatever
        
        # 证书文件
        certificate_file = ${certdir}/server.pem
        
        # CA证书文件
        ca_file = ${certdir}/ca.pem
        
        # DH参数文件
        dh_file = ${certdir}/dh
        
        # 随机数源
        random_file = /dev/urandom
        
        # 是否要求客户端证书
        # client_cert = yes
        
        # 证书验证深度
        # verify_depth = 0
        
        # CRL检查
        # check_crl = no
        
        # 证书生命周期检查
        # check_cert_valid = yes
        
        # 允许的TLS版本
        # tls_min_version = "1.0"
        # tls_max_version = "1.3"
        
        # 加密套件
        # cipher_list = "HIGH"
    }
    
    # EAP-TLS配置
    tls {
        tls = tls-common
        
        # 虚拟服务器配置
        # virtual_server = check-eap-tls
    }
    
    # EAP-TTLS配置
    ttls {
        tls = tls-common
        
        # 默认内层认证
        default_eap_type = mschapv2
        
        # 是否复制属性到内层请求
        # copy_request_to_tunnel = no
        
        # 是否使用隧道化回复属性
        # use_tunneled_reply = no
        
        # 内层虚拟服务器
        virtual_server = inner-tunnel
    }
    
    # EAP-PEAP配置
    peap {
        tls = tls-common
        
        # 默认内层认证
        default_eap_type = mschapv2
        
        # 是否复制属性到内层请求
        # copy_request_to_tunnel = no
        
        # 是否使用隧道化回复属性
        # use_tunneled_reply = no
        
        # 内层虚拟服务器
        virtual_server = inner-tunnel
    }
    
    # EAP-MSCHAPv2配置
    mschapv2 {
        # 是否发送EAP-Success
        send_error = yes
        
        # 是否使用NT域
        use_mppe = yes
        require_encryption = yes
        require_strong = yes
    }
}
```

### 虚拟服务器配置

**目录**: [raddb/sites-available/](file:///workspace/raddb/sites-available/)

**default虚拟服务器结构**:

```
server default {
    # 协议命名空间
    namespace = radius
    
    # 日志配置（可选）
    # log = some_other_logging_destination
    
    # RADIUS协议特定配置
    radius {
        # Access-Request特定配置
        Access-Request {
            # 会话管理（主要用于EAP）
            session {
                # 最大会话数
                # max = 4096
                
                # 最大轮次
                # max_rounds = 40
                
                # 会话超时（秒）
                # timeout = 15
                
                # 去重键
                # dedup_key = Calling-Station-Id
            }
        }
    }
    
    # 本地字典定义
    dictionary {
        # string my_attribute
    }
    
    # 监听配置
    listen {
        type = auth
        ipaddr = *
        port = 1812
        # proto = udp
        # interface = eth0
    }
    
    listen {
        type = acct
        ipaddr = *
        port = 1813
    }
    
    # 主要处理阶段
    authorize { ... }
    authenticate { ... }
    preacct { ... }
    accounting { ... }
    post-auth { ... }
}
```

**主要处理阶段说明**:

1. **recv Access-Request**: 接收认证请求，初始化处理
2. **authorize**: 授权前检查，收集用户信息，选择认证方式
3. **authenticate**: 执行实际的认证
4. **post-auth**: 认证后处理，添加回复属性，发送响应
5. **recv Accounting-Request**: 接收计费请求
6. **preacct**: 计费前处理，验证计费请求
7. **accounting**: 记录计费信息
8. **session**: 会话管理

### 配置启用方法

模块启用:
```bash
cd /etc/raddb/mods-enabled
ln -s ../mods-available/sql
ln -s ../mods-available/ldap
```

虚拟服务器启用:
```bash
cd /etc/raddb/sites-enabled
ln -s ../sites-available/default
```

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
