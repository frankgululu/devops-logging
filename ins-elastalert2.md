# elastalert2 安装部署
```
helm repo add jertel https://helm.jertel.com
```
## 安装chart, 其名字是elastalert2
```
helm install elastalert2 jertel/elastalert2
```

## 离线环境安装
```
# 在有网络的环境里添加repo，下载chart包
helm repo add jertel https://helm.jertel.com
helm pull jertel/elastalert2
# 解压下载之后的chart包
tar -zxvf elastalert2-2.1.1.tgz
# 修改对应的values.yaml
下面具体会说
# 安装部署
helm install elastalert2 ./elastalert2 -n logging
```
## value.yaml 修改
### 配置es 连接
由于es开启了认证，所以我们需要先查看es的secret 的名称

```yaml
elasticsearch:
  # elasticsearch endpoint e.g. (svc.namespace||svc)
  host: elasticsearch-master # es的service 名称
  # elasticsearch port
  port: 9200
  # whether or not to connect to es_host using TLS
  useSsl: "False"
  # Username if authenticating to ES with basic auth
  username: ""
  # Password if authenticating to ES with basic auth
  password: ""
  # Specifies an existing secret to be used for the ES username/password
  credentialsSecret: "elastic-credentials" # es secret 的名称
  # The key in elasticsearch.credentialsSecret that stores the ES password
  credentialsSecretUsernameKey: "username" # es的secret user 的 key
  # The key in elasticsearch.credentialsSecret that stores the ES username
  credentialsSecretPasswordKey: "password" # es secret password 的key
```
### 配置邮件认证
- 进入到pre_elastalert2目录下，创建以smtp_auth.yaml文件为来源的secret
```
# 注意替换这里的 命名空间为自己需要安装的命名空间
sh -x create-secret.sh
```
- 以volume的方式将secret挂砸进来
```yaml
extraVolumes:
   - name: smtp-auth
     secret:
       secretName: elastalert-smtp-auth
       items:
         - key: smtp_auth.yaml
           path: smtp_auth.yaml
           mode: 0777

extraVolumeMounts:
   - name: smtp-auth
     mountPath: /mnt/smtp_auth.yaml
     subPath: smtp_auth.yaml
     readOnly: true
```
### 配置elastalert2 全局配置
```yaml
# 配置全局config

apiVersion: v1
kind: Secret
metadata:
  name: elastalert-config-secret
type: Opaque
stringData:
   elastalert_config: |-
     rules_folder: /opt/elastalert/rules
     scan_subdirectories: false
     run_every: # ElastAlert 多久query 一次ES
       minutes: 5
     buffer_time:
       minutes: 15
     es_host: elasticsearch-master
     es_port: 9200
     writeback_index: elastalert
     use_ssl: False
     verify_certs: True
     alert_time_limit:
```
### 编写告警规则以及告警方式
```yaml
rules:
   deadman_email: |-
     ---
     # 告警规则rule
     name: Blockchain Error Log --test
     type: frequency # 这个类型和num_events配合使用
     index: filebeat-containers-*
     num_events: 10 # 4小时之内命中10次，就算触发一次告警规则
     timeframe:
       hours: 4
     filter:
     - query:
         query_string:
           query: "message: ERROR OR message: error"

     aggregation:
       minutes: 5  #聚合5分钟
       #schedule: '*/5 * * * *' #也可以聚合之后，每隔多久以cron的形式发送
     aggregation_key: 'kubernetes.pod.name' #按照某个字段进行聚合，aggreation_key 会和rule的名称拼接在一起作为一个组，单独发送告警，相同的message是一个组
```
### 配置邮件告警方式已经邮件内容格式
```yaml
    # 配置alert内容格式
     alert_subject: "[测试环境] 错误日志发生在 {1} @ {0}"
     alert_subject_args: ["kubernetes.namespace","kubernetes.pod.name"]
     alert_text_type: alert_text_only
     alert_text: |

      > [时间]: {0}

      > [日志索引]: {1}

      > [命名空间]: {2}

      > [Pod容器]: {3}

      > [错误消息]: {4}
     alert_text_args: [ "@timestamp","_index","kubernetes.namespace","kubernetes.pod.name","message" ]

     # 配置告警形式为email
     alert:
     - "email"
     email:
     - xxxxxxg@xxxxxx.com
     - xxxxx@xxxxx.com
     - xxxxe@xxxxx.com
     smtp_host: 'smtp.mxhichina.com'
     smtp_port: 80
     smtp_auth_file: /mnt/smtp_auth.yaml
     email_reply_to: xxxxr@xxxx.com
     from_addr: xxxx@xxxx.com
```
## 使修改生效
```
helm upgrade elaster ./elastalert2 -n xxxx
```

## reference:
[https://elastalert2.readthedocs.io/en/latest/elastalert.html](https://elastalert2.readthedocs.io/en/latest/elastalert.html)

[https://github.com/jertel/elastalert2](https://github.com/jertel/elastalert2)
