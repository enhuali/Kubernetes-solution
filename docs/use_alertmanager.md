# Alertmanager使用

## 配置邮件报警

通过此文件控制报警路由规则

```
// 进入operator工作目录
cd /etc/ansible/extension/monitoring

// 新增alertmanager-routes.yaml文件
vim alertmanager-routes.yaml
alertmanager:
  config:
    global:
			......略
        
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      smtp_smarthost: 'smtp.exmail.qq.com:465'
      smtp_from: 'msg@qq.com'
      smtp_auth_username: 'msg@qq.com'
      smtp_auth_password: 'sad12s'
      smtp_require_tls: false
    templates:
      - "/etc/alertmanager-tmpl/wechat.tmpl"
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      group_interval: 1m
      repeat_interval: 4h
      receiver: default
      routes:
      - receiver: email
        group_wait: 10s
        match:
          alertname: Watchdog
    receivers:
    - name: 'default'
      email_configs:
      - to: '1262735860@qq.com'
        send_resolved: true
    - name: 'email'
      email_configs:
      - to: '1poypox@163.com'
        send_resolved: true
```

- 加载配置文件

  ```
  helm upgrade  promethues-operator --namespace monitoring -f prome-settings.yaml -f prome-rules.yaml -f prome-servicemonitors.yaml -f alertmanager-routes.yaml prometheus-operator
  ```

## 报警时长设置

首先警报的激活是由Prometheus Server的Alert Rule引起的，Alert Rule可以设置阀值，当达到rule设置的阀值时发送警报给Alertmanager。所以警报具体的发送时间跟Prometheus和Alertmanager的配置密切相关。这里有几个关键的配置。
**Prometheus相关**:

```
1. scrape_interval: How frequently to scrape targets default=1m。Server端抓取数据的时间间隔
2. scrape_timeout: How long until a scrape request times out. default = 10s 数据抓取的超时时间
3. evaluation_interval: How frequently to evaluate rules. default = 1m 评估报警规则的时间间隔
```

**Alertmanger相关：**

```
1. group_wait： How long to initially wait to send a notification for a group of alerts. Allows to wait for an inhibiting alert to arrive or collect more initial alerts for the same group. (Usually ~0s to few minutes. default = 30s)发送一组新的警报的初始等待时间,也就是初次发警报的延时
2. group_interval：How long to wait before sending a notification about new alerts that are added to a group of alerts for which an initial notification has already been sent. (Usually ~5m or more. default = 5m)初始警报组如果已经发送，需要等待多长时间再发送同组新产生的其他报警
3. repeat_interval: How long to wait before sending a notification again if it has already been sent successfully for an alert (Usually ~3h or more. default = 4h ) 如果警报已经成功发送，间隔多长时间再重复发送
```

**另外说下Alert的三种状态：**

```
1. pending：警报被激活，但是低于配置的持续时间。这里的持续时间即rule里的FOR字段设置的时间。改状态下不发送报警。
2. firing：警报已被激活，而且超出设置的持续时间。该状态下发送报警。
3. inactive：既不是pending也不是firing的时候状态变为inactive
```

需要注意一点：**只有在评估周期期间，警报才会从当前状态转移到另一个状态。**

## 注意事项

- 邮件报警错误日志

  Notify for alerts failed" num_alerts=6 err="*notify.loginAuth failed: 535 Error: authentication failed
  这是由于密码需要使用客户端授权码，而非账户密码
