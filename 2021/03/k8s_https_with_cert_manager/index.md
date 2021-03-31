# k8s https证书管理（cert-manager）


> 当前说明文档基于[cert-manager Documentation v1.2](https://cert-manager.io/docs/configuration/acme/)

### 1. 安装

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
```

### 2. 配置颁发证书机构

> 这里仅仅说明ACME配置HTTP01 方式, DNS01是采用添加DNS txt记录方式

```yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: support@example.com #填写相关的邮箱地址
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: letsencrypt-issuer-example-key 
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: edge-nginx #这里根据自身k8s选择正确的 ingress class
          
```

### 3. 使用

##### 步骤
1. 配置服务（k8s service)
2. 配置外部域名解析配置到ingress 边缘节点。 此例子中`ingress class` 为 `edge-nginx`
3. 配置ingress


##### 示例

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    # add an annotation indicating the issuer to use. 
    # 这里的名字必须跟第二节中配置的颁发者名对上
    cert-manager.io/cluster-issuer: letsencrypt-prod
  name: myIngress
  namespace: myIngress
spec:
  rules:
  - host: dev.example.com
    http:
      paths:
      - backend:
          serviceName: myservice
          servicePort: 80
        path: /
  tls: # < placing a host in the TLS config will indicate a certificate should be created
  - hosts:
    - dev.example.com
    secretName: myingress-cert # < cert-manager will store the created certificate in this secret.
```

一

查看证书颁发情况

```output
[root@mylinux ~]# kubectl describe certificate dev-example-tls -n example
Name:         dev-example-tls
Namespace:    example
Labels:       app=example-web-ingress
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
...
Spec:
  Dns Names:
    dev.example.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  dev-example-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2021-03-31T02:31:52Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-06-29T01:32:25Z
  Not Before:              2021-03-31T01:32:25Z
  Renewal Time:            2021-05-30T01:32:25Z
  Revision:                1
Events:
  Type    Reason     Age        From          Message
  ----    ------     ----       ----          -------
  Normal  Issuing    <invalid>  cert-manager  Issuing certificate as Secret was previously issued by ClusterIssuer.cert-manager.io/letsencrypt-staging
  Normal  Reused     <invalid>  cert-manager  Reusing private key stored in existing Secret resource "dev-example-tls"
  Normal  Requested  <invalid>  cert-manager  Created new CertificateRequest resource "dev-example-tls-kq2pn"
  Normal  Issuing    <invalid>  cert-manager  The certificate has been successfully issued
[root@mylinux ~]# 

```

### 问题

1. 证书颁发异常，报错如下

```
Events:
  Type     Reason     Age        From          Message
  ----     ------     ----       ----          -------
  Normal   Issuing    <invalid>  cert-manager  Issuing certificate as Secret does not exist
  Normal   Generated  <invalid>  cert-manager  Stored new private key in temporary Secret resource "dev-example-tls-z2pbs"
  Normal   Requested  <invalid>  cert-manager  Created new CertificateRequest resource "dev-example-tls-jgz6j"
  Warning  Failed     <invalid>  cert-manager  The certificate request has failed to complete and will be retried: Failed to wait for order resource "dev-example-tls-jgz6j-684099693" to become ready: order is in "invalid" state:
```
此问题由于外部解析没有异常导致，在颁发证书时需要DNS解析可以解析到当前的边界节点，acme会发起http请求校验。

### 其他可能用到的命令

```bash
kubectl describe ingress -n example #查看ingress 状态
```

### 参考

https://cert-manager.io/docs/installation/kubernetes/


