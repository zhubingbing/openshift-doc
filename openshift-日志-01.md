# openshift-elk

https://github.com/openshift/origin-aggregated-logging


# fluentd
提供搜集日志服务
dockerfile: https://github.com/openshift/origin-aggregated-logging/blob/master/fluentd/Dockerfile


1. fluentd配置文件
bash-4.2# tree
.
|-- configs.d
|   |-- dynamic
|   |   `-- input-docker-default-docker.conf
|   |-- filter-k8s-meta-for-mux-client.conf
|   |-- filter-post-mux.conf
|   |-- filter-post-z-mux-client.conf
|   |-- filter-post-z-retag-one.conf
|   |-- filter-post-z-retag-two.conf
|   |-- filter-pre-a-audit-exclude.conf
|   |-- filter-pre-mux-client.conf
|   |-- filter-pre-mux.conf
|   |-- input-post-forward-mux.conf
|   |-- input-pre-audit-log.conf
|   |-- input-pre-debug.conf
|   |-- input-pre-monitor.conf
|   |-- openshift
|   |   |-- README
|   |   |-- filter-exclude-journal-debug.conf
|   |   |-- filter-k8s-meta.conf
|   |   |-- filter-k8s-record-transform.conf
|   |   |-- filter-kibana-transform.conf
|   |   |-- filter-post-genid.conf
|   |   |-- filter-post-z-retag-one.conf
|   |   |-- filter-pre-a-audit-exclude.conf
|   |   |-- filter-pre-force-utf8.conf
|   |   |-- filter-retag-journal.conf
|   |   |-- filter-syslog-record-transform.conf
|   |   |-- filter-viaq-data-model.conf
|   |   |-- input-pre-audit-log.conf
|   |   |-- input-pre-systemd.conf
|   |   |-- output-applications.conf
|   |   |-- output-es-config.conf
|   |   |-- output-es-ops-config.conf
|   |   |-- output-operations.conf
|   |   `-- system.conf
|   `-- user
|       |-- fluent.conf -> ..data/fluent.conf
|       |-- secure-forward.conf -> ..data/secure-forward.conf
|       `-- throttle-config.yaml -> ..data/throttle-config.yaml
|-- fluent.conf -> /etc/fluent/configs.d/user/fluent.conf
|-- keys
|   |-- ca -> ..data/ca
|   |-- cert -> ..data/cert
|   |-- key -> ..data/key
|   |-- ops-ca -> ..data/ops-ca
|   |-- ops-cert -> ..data/ops-cert
|   `-- ops-key -> ..data/ops-key
`-- plugin
    |-- filter_k8s_meta_for_mux_client.rb
    |-- out_syslog.rb
    |-- out_syslog_buffered.rb
    |-- parser_viaq_docker_audit.rb
    `-- viaq_docker_audit.rb


# elastaticserach
dockerfile：https://github.com/openshift/origin-aggregated-logging/blob/master/elasticsearch/Dockerfile

有2个容器
1. es容器： 提供elasticsearch服务

elasticsearch: 配置信息

es配置文件路径（服务设置）：/usr/share/java/elasticsearch/config（pod容器里面）

es（索引和切词一些设置）： /usr/share/elasticsearch
es 存储的目录（rw）： /elasticsearch/persistent
es serviceacount: /var/run/secrets/kubernetes.io/serviceaccount


2. proxy容器： oauth-proxy


1. 向OpenShift OAuth服务器或Kubernetes主机提供身份验证和授权；
```
 --upstream-ca=/etc/elasticsearch/secret/admin-ca
      --https-address=:4443
      -provider=openshift
      -client-id=system:serviceaccount:logging:aggregated-logging-elasticsearch
      -client-secret-file=/var/run/secrets/kubernetes.io/serviceaccount/token
      -cookie-secret=U2VvRmRxdlU4MDNMZ0hVOQ==
      -basic-auth-password=Eqhq7hNAsrh2oFda
      -upstream=https://localhost:9200
      -openshift-sar={"namespace": "logging", "verb": "view", "resource": "prometheus", "group": "metrics.openshift.io"}
      -openshift-delegate-urls={"/": {"resource": "prometheus", "verb": "view", "group": "metrics.openshift.io", "namespace": "logging"}}
      --tls-cert=/etc/tls/private/tls.crt
      --tls-key=/etc/tls/private/tls.key
      -pass-access-token
      -pass-user-headers


一些参数解析：

-- provider : OAuth provider 默认是google
--openshift-sar=JSON: 需要特定的权限才能通过OAuth进行登录

SAR代表“Subject Access Review”，它是发送给OpenShift或Kubernetes服务器的请求，用于检查特定用户的访问权限。期望单个主题访问审核JSON对象或JSON数组，所有这些必须满足用户才能访问后端服务器。

优点：

使用OAuth流程保护整个网站或API的最简单方法
不需要授予代理服务帐户的额外权限
缺点：

不太适合服务到服务的访问
上游服务器的全或全无保护
例子:
```
# 如果用户可以在在logging namespace中查看（view）资源prometheus 组为metrics.openshift.io 则容许访问
 -openshift-sar={"namespace": "logging", "verb": "view", "resource": "prometheus", "group": "metrics.openshift.io"}
```
访问代理的用户将被重定向到使用OpenShift进行的OAuth登录，并且必须授予代理访问权限才能查看他们的用户信息并请求他们的权限。一旦他们将该权利授予代理，它将检查用户是否具有所需的权限。如果他们不这样做，他们将被授予拒绝错误的权限。如果是，他们将通过cookie登录。

运行oc explain subjectaccessreview 查看字段详细解释

-openshift-delegate-urls： 将身份验证和授权委托给OpenShift基础架构
OpenShift为最终用户和服务帐户使用承载令牌。运行基础结构服务时，将所有身份验证和授权委派给主服务器可能更容易。该--openshift-delegate-urls=JSON标志启用委派，要求主方验证任何传入请求的Authorization: Bearer头部或客户端证书将被转发给主服务器进行验证。

例子：
```
# 如果提供的令牌可以在logging命名空间中有查看prometheus资源的权限，则允许访问。
-openshift-delegate-urls={"/": {"resource": "prometheus", "verb": "view", "group": "metrics.openshift.io", "namespace": "logging"}}
```

--client-id 和 --client-secret:
读取--client-id和--client-secret通过OpenShift注入的服务帐户信息。使用的值/var/run/secrets/kubernetes.io/serviceaccount/namespace ，以构建正确的--client-id，并且内容 /var/run/secrets/kubernetes.io/serviceaccount/token的--client-secret

--upstream和upstream-ca:  上游服务和访问上游的证书

# kibana
有2个容器
1. kibana服务: 


2. kiban-proxy: origin-logging-auth-proxy

为了根据OpenShift的Oauth2对Kibana用户进行身份验证，需要在Kibana前面运行一个代理
origin-logging-auth-proxy就是实现这个的功能的代理（一个kibana插件）：

https://github.com/fabric8io/openshift-auth-proxy/tree/88e70c0f5d179e266300b3ddc223f9c861acf2d2


# 部署

https://docs.openshift.org/latest/install_config/aggregate_logging.html

在inventory文件中设置:
```
openshift_logging_install_logging=true
```

执行如下命令部署

```
 ansible-playbook -i inventory/xxx(自己定义的文件) playbooks/openshift-logging/config.yml
```

# 服务访问流程

