# 使用Graylog收集AD审核日志

## AD审核策略

### 配置审核策略

**注意： 配置审核策略只操作一次，其它AD服务器不需要再次执行。**

参考[Audit Policy Recommendations](https://docs.microsoft.com/zh-cn/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations)修改组策略。

修改完成后，使用命令"auditpol /get /category:*"检查设置结果。

```
C:\>auditpol /get /category:*
系统审核策略
类别/子类别                                    设置
系统
  安全系统扩展                                  成功和失败
  系统完整性                                   成功和失败
  IPsec 驱动程序                              成功和失败
  其他系统事件                                  无审核
  安全状态更改                                  成功和失败
登录/注销
  登录                                      成功和失败
  注销                                      成功
  帐户锁定                                    成功
  IPsec 主模式                               无审核
  IPsec 快速模式                              无审核
  IPsec 扩展模式                              无审核
  特殊登录                                    成功和失败
  其他登录/注销事件                               成功和失败
  网络策略服务器                                 无审核
对象访问
  文件系统                                    无审核
  注册表                                     无审核
  内核对象                                    无审核
  SAM                                     无审核
  证书服务                                    无审核
  已生成应用程序                                 无审核
  句柄操作                                    无审核
  文件共享                                    无审核
  筛选平台数据包丢弃                               无审核
  筛选平台连接                                  无审核
  其他对象访问事件                                无审核
  详细的文件共享                                 无审核
特权使用
  敏感权限使用                                  无审核
  非敏感权限使用                                 无审核
  其他权限使用事件                                无审核
详细追踪
  进程终止                                    无审核
  DPAPI 活动                                成功和失败
  RPC 事件                                  无审核
  进程创建                                    成功和失败
策略改动
  审核策略更改                                  成功和失败
  身份验证策略更改                                成功和失败
  授权策略更改                                  无审核
  MPSSVC 规则级别策略更改                         成功
  筛选平台策略更改                                无审核
  其他策略更改事件                                无审核
帐户管理
  用户帐户管理                                  成功和失败
  计算机帐户管理                                 成功和失败
  安全组管理                                   成功和失败
  分发组管理                                   无审核
  应用程序组管理                                 无审核
  其他帐户管理事件                                成功和失败
DS 访问
  目录服务更改                                  成功和失败
  目录服务复制                                  无审核
  详细的目录服务复制                               无审核
  目录服务访问                                  成功和失败
帐户登录
  Kerberos 服务票证操作                         成功和失败
  其他帐户登录事件                                成功和失败
  Kerberos 身份验证服务                         成功和失败
  凭据验证                                    成功和失败
```

## Graylog系统收集AD审核日志

### 在AD服务器安装graylog sidecars客户端

1. 创建Token

    在graylog服务器创建用于访问graylog服务api的token。

    导航到http://<graylog>/system/authentication/users/tokens/admin

    | graylog | token |
    | ------ | ------ |
    | 1 |  |
    



2. 安装：

    下载sidecar：https://github.com/Graylog2/collector-sidecar/releases

    运行graylog_sidecar_installer_1.0.2-1.exe，执行安装。

    安装过程中填写Graylog API的URL和访问API的token。

    | graylog | URL |
    | ------ | ------ |
    | 1 | http://graylog:9000/api |

3. 注册系统服务：

    ```
    "C:\Program Files\graylog\sidecar\graylog-sidecar.exe" -service install
    "C:\Program Files\graylog\sidecar\graylog-sidecar.exe" -service start
    ```

### 在Graylog服务器端执行以下操作

1. 创建Winlogbeat输入

    点击System > Inputs, 选择“Beats”然后点击“Launch new input”。
    
    | item | value |
    | ------ | ------ |
    | Global | checked |    
    | Title | winad winlogbeat |
    | Bind address | 0.0.0.0 |    
    | Port | 5055 |    
    | 其它 | 默认值 |    


2. 配置采集器

    点击System > Sidecars, 导航到Sidecars Overview页面。

    在此页面点击“Configuration”,然后点击“Create Configuration”。 

    | item | value |
    | ------ | ------ |
    | Name | winad winlogbeat |    
    | Collector | winlogbeat on Windows |
    | Configuration |见下面|    
    
    ```
    # Needed for Graylog
    fields_under_root: true
    fields.collector_node_id: ${sidecar.nodeName}
    fields.gl2_source_collector: ${sidecar.nodeId}

    output.logstash:
    hosts: ["graylog:5055"]
    path:
    data: C:\Program Files\Graylog\sidecar\cache\winlogbeat\data
    logs: C:\Program Files\Graylog\sidecar\logs
    tags:
    - winadaudit
    winlogbeat:
    event_logs:
    - name: Security
        processors:
            - drop_event.when.not.or:
                - equals.event_id: 4624
                - equals.event_id: 4625
                - equals.event_id: 4771
                - equals.event_id: 4776
                - equals.event_id: 5136
                - equals.event_id: 5137
                - equals.event_id: 5138
                - equals.event_id: 5139
                - equals.event_id: 5141
                - equals.event_id: 4720
                - equals.event_id: 4723
                - equals.event_id: 4724
                - equals.event_id: 4725
                - equals.event_id: 4726
                - equals.event_id: 4767
                - equals.event_id: 4780
                - equals.event_id: 4781
                - equals.event_id: 4741
                - equals.event_id: 4742
                - equals.event_id: 4743
                - equals.event_id: 4727
                - equals.event_id: 4728
                - equals.event_id: 4729
                - equals.event_id: 4730
                - equals.event_id: 4730
                - equals.event_id: 4731
                - equals.event_id: 4732
                - equals.event_id: 4733
                - equals.event_id: 4754
                - equals.event_id: 4756
                - equals.event_id: 4757
                - equals.event_id: 4758
        ignore_older: 48h
    ```

    注意： 台北graylog的output.logstash hosts设置为台北graylog服务器ip地址。

    
3. 创建索引集

    点击System > Indices, 导航到Indices & Index Sets页面。

    | item | value |
    | ------ | ------ |
    | Title | WinAD Audit index set |    
    | Description | WinAD Audit index set |
    | Index prefix |winadaudit_|    
    | Index rotation strategy |Index Time|  
    | Rotation period |P2M|  
    | Select retention strategy |Delete|  
    | Max number of indices |10|  

4. 创建Stream

    点击Streams，然后点击“Create Stream”

    | item | value |
    | ------ | ------ |
    | Title | WinAD Audit stream |
    | Description | Field winlogbeat_tags must contain winadaudit |
    | Index Set | WinAD Audit index set |
    | Remove matches from ‘All messages’ stream | Checked |

    配置Rules，符合规则“字段 winlogbeat_tags 包含 winadaudit”的日志路由到此流中。

    | item | value |
    | ------ | ------ |
    | Field | winlogbeat_tags |
    | Type | contain  |
    | Value | winadaudit |
    
5. 创建视图

    创建相关视图。

    | Index | Title |
    | ------ | ------ |
    | 1 | WinAD Audit Logon |
    | 2 | WinAD Audit Directory Object |
    | 3 | WinAD Audit User Account Management |
    | 4 | WinAD Audit Computer Account Management |
    | 5 | WinAD Audit Security Group |

    **1 WinAD Audit Logon**

    | name | sql |
    | ------ | ------ |
    | 本地交互登录 | winlogbeat_event_id:(4624 OR 4625) AND winlogbeat_event_data_LogonType:2 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | 网络登录 | winlogbeat_event_id:(4624 OR 4625) AND winlogbeat_event_data_LogonType:3 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | RDP登录 | winlogbeat_event_id:(4624 OR 4625) AND winlogbeat_event_data_LogonType:10 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | 本地缓存登录 | winlogbeat_event_id:(4624 OR 4625) AND winlogbeat_event_data_LogonType:11 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | 尝试验证帐户的凭据 | winlogbeat_event_id:4776 AND NOT winlogbeat_event_data_Status:0x0 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | 单账号多IP登录 | winlogbeat_event_id:4624 AND winlogbeat_event_data_LogonType:3 AND NOT winlogbeat_event_data_IpAddress:- AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | 非上班时间登录 | 未实现 |

    **2 WinAD Audit Directory Object**

    | name | sql |
    | ------ | ------ |
    | 已创建目录服务对象 | winlogbeat_event_id:5137 AND winlogbeat_event_data_ObjectClass:(computer OR user) |
    | 已修改目录服务对象 | winlogbeat_event_id:5136 AND winlogbeat_event_data_ObjectClass:(computer OR user) |
    | 已移动目录服务对象 | winlogbeat_event_id:5139 AND winlogbeat_event_data_ObjectClass:(computer OR user) |
    | 已删除目录服务对象 | winlogbeat_event_id:5141 AND winlogbeat_event_data_ObjectClass:(computer OR user) |

    **3 WinAD Audit User Account Management**

    | name | sql |
    | ------ | ------ |
    | 已创建用户帐户 | winlogbeat_event_id:4720 |
    | 尝试修改重置密码 | winlogbeat_event_id:(4723 OR 4724) |
    | 已禁用用户帐户 | winlogbeat_event_id:4725 |
    | 已删除用户帐户 | winlogbeat_event_id:4726 |
    | 已解锁用户帐户  | winlogbeat_event_id:4767 |
    | administrators groups | winlogbeat_event_id:4780 |
    | 更改了帐户名称 | winlogbeat_event_id:4781 |

    **4 WinAD Audit Computer Account Management**
    
    | name | sql |
    | ------ | ------ |
    | 已创建计算机帐户 | winlogbeat_event_id:4741 |
    | 已更改计算机帐户 | winlogbeat_event_id:4742 |
    | 已删除计算机帐户 | winlogbeat_event_id:4743 |

    **5 WinAD Audit Security Group**
    
    | name | sql |
    | ------ | ------ |
    | 已创建安全组 | winlogbeat_event_id:(4727 AND 4731 AND 4754) |
    | 成员加入安全组  | winlogbeat_event_id:(4728 AND 4732 AND 4756) |
    | 成员移出安全组  | winlogbeat_event_id:(4729 AND 4733 AND 4757) |
    | 已删除安全组  | winlogbeat_event_id:(4730 AND 4734 AND 4758) |

6. 创建事件告警通知

    3 配置graylog服务器启用邮件告警通知

    vi /etc/graylog/server/server.conf

    ```
    transport_email_enabled = true
    transport_email_hostname = smtp.puhua.net
    transport_email_port = 25
    transport_email_use_auth = true
    transport_email_auth_username = jacky@puhua.net 
    transport_email_auth_password = <密码>
    transport_email_subject_prefix = [graylog]
    transport_email_from_email = jacky@puhua.net 
    ```

    重新启动graylog服务

    ```
    systemctl restart graylog
    ```

    2 点击System > Alerts, 导航到Alerts & Events页面

    在此页面点击“Notifications”,然后点击“Create Notification”。

    | item | value |
    | ------ | ------ |        
    | Title | Email Notification |
    | Notification Type | Email Notification |
    | Sender | jacky@puhua.net  |
    | Subject | [Graylog]${event_definition_title} |
    | User recipient(s) | 相关收件人 |
    | 其它 | 默认值 |

    3 点击“Event Definitions”,然后点击“Create Event Definition”。创建相关事件。

    | item | value |
    | ------ | ------ |    
    | Title | **绕过堡垒机进行RDP远程登录** |
    | Description  | 每隔5分钟，查询5分钟内是否有RDP登录成功（源IP地址不是Jumpserver）的日志，如有则告警。 |
    | Condition Type | Filter & Aggregation |
    | Search Query | winlogbeat_event_id:(4624 OR 4625) AND winlogbeat_event_data_LogonType:10 AND NOT winlogbeat_event_data_IpAddress:jumpserver |
    | Streams  | WinAD Audit stream |
    | Search within the last | 5m |
    | Execute search every | 5m |
    | Create Events for Definition if... | Filter has results |
    | Notification | Email Notification	 |

    | item | value |
    | ------ | ------ |    
    | Title | **域帐户的身份验证失败超过阈值 by Account** |
    | Description | 每隔15分钟查询一次。查询15分钟内域账号登录发生错误的日志，按Account名称聚合，如果记录数超过15，则告警。 |
    | Condition Type | Filter & Aggregation |
    | Search Query | winlogbeat_event_id:4776 AND NOT winlogbeat_event_data_Status:0x0 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | Streams  | WinAD Audit stream |
    | Search within the last | 15m |
    | Execute search every | 15m |
    | Create Events for Definition if... | count(winlogbeat_event_data_TargetUserName) > 15 |
    | Notification | Email Notification	 |


    | item | value |
    | ------ | ------ |    
    | Title | **网络登录失败次数超过阈值 by Account** |
    | Description | 每隔15分钟查询一次。查询15分钟内登录失败且登录类型为网络登录的日志，按Account聚合，如果记录数超过15，则告警。 |
    | Condition Type | Filter & Aggregation |
    | Search Query | winlogbeat_event_id:4625 AND winlogbeat_event_data_LogonType:3 AND NOT winlogbeat_event_data_TargetUserName:*$  |
    | Streams  | WinAD Audit stream |
    | Search within the last | 15m |
    | Execute search every | 15m |
    | Create Events for Definition if... | count(winlogbeat_event_data_TargetUserName) > 15 |
    | Notification | Email Notification	 |

    | item | value |
    | ------ | ------ |    
    | Title | **活动目录异常操作** |
    | Description | 每隔15分钟查询一次。查询15分钟内域对象的创建、修改、移动和删除操作的记录，按对象类型聚合，如果数量超过15，则告警。 |
    | Condition Type | Filter & Aggregation |
    | Search Query | winlogbeat_event_id:(5136 OR 5137 OR 5139 OR 5141)  AND winlogbeat_event_data_ObjectClass:(computer OR user)  |
    | Streams  | WinAD Audit stream |
    | Search within the last | 15m |
    | Execute search every | 15m |
    | Create Events for Definition if... | count(winlogbeat_event_data_ObjectClass) > 15 |
    | Notification | Email Notification	 |   






