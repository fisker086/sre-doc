# altermanager配置详解

```bash
# global 全局配置
global:
  # 当告警的状态有firing变为resolve的以后还要呆多长时间，才宣布告警解除。这个主要是解决某些监控指标在阀值边缘上波动，一会儿好一会儿不好。
  resolve_timeout: 1h
  
  # 邮件告警发件配置
  smtp_smarthost: 'smtp.exmail.qq.com:25'     #stmp服务器主机
  smtp_from: 'dukuan@xxx.com'         
  smtp_auth_username: 'dukuan@xxx.com'
  smtp_auth_password: 'DKxxx'
  # HipChat告警配置
  # hipchat_auth_token: '123456789'
  # hipchat_auth_url: 'https://hipchat.foobar.org/'

  # wechat
  wechat_api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
  wechat_api_secret: 'JJ'  # 应用管理中自建小程序的Secret
  wechat_api_corp_id: 'ww'  # 企业信息中的企业ID

  #配置告警接收者信息
receivers:
    #配置邮件收件人
  - name: 'team-ops-mails'    #起个名字，代称方便实用
    email_configs:            #邮件接收配置
      - to: 'dukuan@xxx.com'  #配置收件人邮箱

    #配置微信收件人
  - name: 'wechat'      #起个名字方便使用
    wechat_configs:     #微信接收配置
      - send_resolved: true       # 为题解决了是否通知
        corp_id: 'ww'             # 企业信息中的企业ID
        api_secret: 'JJ'          # 应用管理中自建小程序的Secret
        to_party: '2'             # 通知组id
        to_user:                  # 通知用户账号
        agent_id: '1000002'       # 应用管理自建小程序的Agentld

      #通过webhook告警,想要通过钉钉等应用告警时配置
  - name: magpie.ding
    webhook_configs:
      - url: http://10.252.3.10:9002/ding_message
        send_resolved: false      # 当问题解决了是否也要通知一下


  # 自定义告警通知模板
templates:
- '/etc/alertmanager/config/*.tmpl'

# route用来设置报警的分发策略，是个重点，告警内容从这里进入，寻找自己应该用那种策略发送出去
route:
  # 告警应该根据那些标签进行分组
  group_by: ['job', 'altername', 'cluster', 'service','severity']

  # 同一组的告警发出前要等待多少秒，这个是为了把更多的告警一个批次发出去
  group_wait: 30s

  #同一组的多批次告警间隔多少秒后，才能发出
  group_interval: 5m

  # 重复的告警要等待多久后才能再次发出去
  repeat_interval: 12h

  # 一级的receiver，也就是默认的receiver，当告警进来后没有找到任何子节点和自己匹配，就用这个receiver
  receiver: 'wechat'

  # 上述route的配置会被传递给子路由节点，子路由节点进行重新配置才会被覆盖
  # 子路由树
  routes:

  # 用于匹配label。此处列出的所有label都匹配到才算匹配
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: 'wechat'

    # 在带有service标签的告警同时有severity标签时，他可以有自己的子路由，同时具有severity != critical的告警则被发送给接收者team-ops-mails,对severity == critical的告警则被发送到对应的接收者即team-ops-pager
    routes:
    - match:
        severity: critical
      receiver: 'wechat'

  # 比如关于数据库服务的告警，如果子路由没有匹配到相应的owner标签，则都默认由team-DB-pager接收
  - match:
      service: database
    receiver: 'wechat'

  # 我们也可以先根据标签service:database将数据库服务告警过滤出来，然后进一步将所有同时带labelkey为database
  - match:
      severity: critical
    receiver: 'wechat'

# 抑制规则，当出现critical告警时 忽略warning
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  #   equal: ['alertname', 'cluster', 'service']
```



# 邮件告警模板

```bash
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta name="viewport" content="width=device-width" />
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<title>{{ template "__subject" . }}</title>
<style>
/* -------------------------------------
    GLOBAL
    A very basic CSS reset
------------------------------------- */
* {
  margin: 0;
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
  box-sizing: border-box;
  font-size: 14px;
}
img {
  max-width: 100%;
}
body {
  -webkit-font-smoothing: antialiased;
  -webkit-text-size-adjust: none;
  width: 100% !important;
  height: 100%;
  line-height: 1.6em;
  /* 1.6em * 14px = 22.4px, use px to get airier line-height also in Thunderbird, and Yahoo!, Outlook.com, AOL webmail clients */
  /*line-height: 22px;*/
}
/* Let's make sure all tables have defaults */
table td {
  vertical-align: top;
}
/* -------------------------------------
    BODY & CONTAINER
------------------------------------- */
body {
  background-color: #f6f6f6;
}
.body-wrap {
  background-color: #f6f6f6;
  width: 100%;
}
.container {
  display: block !important;
  max-width: 600px !important;
  margin: 0 auto !important;
  /* makes it centered */
  clear: both !important;
}
.content {
  max-width: 600px;
  margin: 0 auto;
  display: block;
  padding: 20px;
}
/* -------------------------------------
    HEADER, FOOTER, MAIN
------------------------------------- */
.main {
  background-color: #fff;
  border: 1px solid #e9e9e9;
  border-radius: 3px;
}
.content-wrap {
  padding: 30px;
}
.content-block {
  padding: 0 0 20px;
}
.header {
  width: 100%;
  margin-bottom: 20px;
}
.footer {
  width: 100%;
  clear: both;
  color: #999;
  padding: 20px;
}
.footer p, .footer a, .footer td {
  color: #999;
  font-size: 12px;
}
/* -------------------------------------
    TYPOGRAPHY
------------------------------------- */
h1, h2, h3 {
  font-family: "Helvetica Neue", Helvetica, Arial, "Lucida Grande", sans-serif;
  color: #000;
  margin: 40px 0 0;
  line-height: 1.2em;
  font-weight: 400;
}
h1 {
  font-size: 32px;
  font-weight: 500;
  /* 1.2em * 32px = 38.4px, use px to get airier line-height also in Thunderbird, and Yahoo!, Outlook.com, AOL webmail clients */
  /*line-height: 38px;*/
}
h2 {
  font-size: 24px;
  /* 1.2em * 24px = 28.8px, use px to get airier line-height also in Thunderbird, and Yahoo!, Outlook.com, AOL webmail clients */
  /*line-height: 29px;*/
}
h3 {
  font-size: 18px;
  /* 1.2em * 18px = 21.6px, use px to get airier line-height also in Thunderbird, and Yahoo!, Outlook.com, AOL webmail clients */
  /*line-height: 22px;*/
}
h4 {
  font-size: 14px;
  font-weight: 600;
}
p, ul, ol {
  margin-bottom: 10px;
  font-weight: normal;
}
p li, ul li, ol li {
  margin-left: 5px;
  list-style-position: inside;
}
/* -------------------------------------
    LINKS & BUTTONS
------------------------------------- */
a {
  color: #348eda;
  text-decoration: underline;
}
.btn-primary {
  text-decoration: none;
  color: #FFF;
  background-color: #348eda;
  border: solid #348eda;
  border-width: 10px 20px;
  line-height: 2em;
  /* 2em * 14px = 28px, use px to get airier line-height also in Thunderbird, and Yahoo!, Outlook.com, AOL webmail clients */
  /*line-height: 28px;*/
  font-weight: bold;
  text-align: center;
  cursor: pointer;
  display: inline-block;
  border-radius: 5px;
  text-transform: capitalize;
}
/* -------------------------------------
    OTHER STYLES THAT MIGHT BE USEFUL
------------------------------------- */
.last {
  margin-bottom: 0;
}
.first {
  margin-top: 0;
}
.aligncenter {
  text-align: center;
}
.alignright {
  text-align: right;
}
.alignleft {
  text-align: left;
}
.clear {
  clear: both;
}
/* -------------------------------------
    ALERTS
    Change the class depending on warning email, good email or bad email
------------------------------------- */
.alert {
  font-size: 16px;
  color: #fff;
  font-weight: 500;
  padding: 20px;
  text-align: center;
  border-radius: 3px 3px 0 0;
}
.alert a {
  color: #fff;
  text-decoration: none;
  font-weight: 500;
  font-size: 16px;
}
.alert.alert-warning {
  background-color: #E6522C;
}
.alert.alert-bad {
  background-color: #D0021B;
}
.alert.alert-good {
  background-color: #68B90F;
}
/* -------------------------------------
    INVOICE
    Styles for the billing table
------------------------------------- */
.invoice {
  margin: 40px auto;
  text-align: left;
  width: 80%;
}
.invoice td {
  padding: 5px 0;
}
.invoice .invoice-items {
  width: 100%;
}
.invoice .invoice-items td {
  border-top: #eee 1px solid;
}
.invoice .invoice-items .total td {
  border-top: 2px solid #333;
  border-bottom: 2px solid #333;
  font-weight: 700;
}
/* -------------------------------------
    RESPONSIVE AND MOBILE FRIENDLY STYLES
------------------------------------- */
@media only screen and (max-width: 640px) {
  body {
    padding: 0 !important;
  }
  h1, h2, h3, h4 {
    font-weight: 800 !important;
    margin: 20px 0 5px !important;
  }
  h1 {
    font-size: 22px !important;
  }
  h2 {
    font-size: 18px !important;
  }
  h3 {
    font-size: 16px !important;
  }
  .container {
    padding: 0 !important;
    width: 100% !important;
  }
  .content {
    padding: 0 !important;
  }
  .content-wrap {
    padding: 10px !important;
  }
  .invoice {
    width: 100% !important;
  }
}
</style>
</head>

<body itemscope itemtype="http://schema.org/EmailMessage">

<table class="body-wrap">
  <tr>
    <td></td>
    <td class="container" width="600">
      <div class="content">
        <table class="main" width="100%" cellpadding="0" cellspacing="0">
          <tr>
            {{ if gt (len .Alerts.Firing) 0 }}
            <td class="alert alert-warning">
            {{ else }}
            <td class="alert alert-good">
            {{ end }}
              {{ .Alerts | len }} alert{{ if gt (len .Alerts) 1 }}s{{ end }} for {{ range .GroupLabels.SortedPairs }}
                {{ .Name }}={{ .Value }} 
              {{ end }}
            </td>
          </tr>
          <tr>
            <td class="content-wrap">
              <table width="100%" cellpadding="0" cellspacing="0">
                <tr>
                  <td class="content-block">
                    <a href='{{ template "__alertmanagerURL" . }}' class="btn-primary">View in {{ template "__alertmanager" . }}</a>
                  </td>
                </tr>
                {{ if gt (len .Alerts.Firing) 0 }}
                <tr>
                  <td class="content-block">
                    <strong>[{{ .Alerts.Firing | len }}] Firing</strong>
                  </td>
                </tr>
                {{ end }}
                {{ range .Alerts.Firing }}
                <tr>
                  <td class="content-block">
                    <strong>Labels</strong><br />
                    {{ range .Labels.SortedPairs }}{{ .Name }} = {{ .Value }}<br />{{ end }}
                    {{ if gt (len .Annotations) 0 }}<strong>Annotations</strong><br />{{ end }}
                    {{ range .Annotations.SortedPairs }}{{ .Name }} = {{ .Value }}<br />{{ end }}
                    <a href="{{ .GeneratorURL }}">Source</a><br />
                  </td>
                </tr>
                {{ end }}

                {{ if gt (len .Alerts.Resolved) 0 }}
                  {{ if gt (len .Alerts.Firing) 0 }}
                <tr>
                  <td class="content-block">
                    <br />
                    <hr />
                    <br />
                  </td>
                </tr>
                  {{ end }}
                <tr>
                  <td class="content-block">
                    <strong>[{{ .Alerts.Resolved | len }}] Resolved</strong>
                  </td>
                </tr>
                {{ end }}
                {{ range .Alerts.Resolved }}
                <tr>
                  <td class="content-block">
                    <strong>Labels</strong><br />
                    {{ range .Labels.SortedPairs }}{{ .Name }} = {{ .Value }}<br />{{ end }}
                    {{ if gt (len .Annotations) 0 }}<strong>Annotations</strong><br />{{ end }}
                    {{ range .Annotations.SortedPairs }}{{ .Name }} = {{ .Value }}<br />{{ end }}
                    <a href="{{ .GeneratorURL }}">Source</a><br />
                  </td>
                </tr>
                {{ end }}
              </table>
            </td>
          </tr>
        </table>

        <div class="footer">
          <table width="100%">
            <tr>
              <td class="aligncenter content-block"><a href='{{ .ExternalURL }}'>Sent by {{ template "__alertmanager" . }}</a></td>
            </tr>
          </table>
        </div></div>
    </td>
    <td></td>
  </tr>
</table>

</body>
</html>
```



# wechat告警模板

```bash
{{ define "wechat.default.message" }}
{{ if gt (len .Alerts.Firing) 0 -}}
Alerts Firing:
{{ range .Alerts }}
告警级别：{{ .Labels.severity }}
告警类型：{{ .Labels.alertname }}
故障主机: {{ .Labels.instance }}
告警主题: {{ .Annotations.summary }}
告警详情: {{ .Annotations.description }}
触发时间: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
{{- end }}
{{- end }}
{{ if gt (len .Alerts.Resolved) 0 -}}
Alerts Resolved:
{{ range .Alerts }}
告警级别：{{ .Labels.severity }}
告警类型：{{ .Labels.alertname }}
故障主机: {{ .Labels.instance }}
告警主题: {{ .Annotations.summary }}
触发时间: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
恢复时间: {{ .EndsAt.Format "2006-01-02 15:04:05" }}
{{- end }}
{{- end }}
告警链接:
{{ template "__alertmanagerURL" . }}
{{- end }}
```

