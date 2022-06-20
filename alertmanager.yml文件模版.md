邮件报警配置
```sql
global:
  smtp_smarthost: 'smtp.qq.com:25'
  smtp_from: ''
  smtp_auth_username: ''
  smtp_auth_password: ''
  smtp_require_tls: false

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5s
  repeat_interval: 10s
  receiver: default-receiver

receivers:
- name: 'default-receiver'
  email_configs:
  - to: 'wxcui@aibee.com'
    html: '{{ template "alert.html" . }}'
    headers: { Subject: "[WARN] 报警邮件test" }
```
