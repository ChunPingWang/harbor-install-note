
Harbor安裝
===


## Table of Contents

[TOC]


憑證準備
---
```gherkin=
openssl genrsa -out ca.key 4096

openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=TW/ST=Taiwan/L=Taipei/O=example/OU=Personal/CN=microservice.tw" \
 -key ca.key \
 -out ca.crt


 openssl genrsa -out microservice.tw.key 4096

 openssl req -sha512 -new \
    -subj "/C=TW/ST=Taiwan/L=Taipei/O=example/OU=Personal/CN=microservice.tw" \
    -key microservice.tw.key \
    -out microservice.tw.csr
```

```gherkin=
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=*.microservice.tw
DNS.2=reg.microservice.tw
DNS.3=microservice.tw
DNS.4=reg
EOF
```
```gherkin=
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in microservice.tw.csr \
    -out microservice.tw.crt
```

```gherkin=
sudo mkdir -p /data/cert
sudo cp microservice.tw.crt /data/cert/
sudo cp microservice.tw.key /data/cert/   
```
```gherkin=
openssl x509 -inform PEM -in microservice.tw.crt -out microservice.tw.cert
```

```gherkin=
sudo mkdir -p /etc/docker/certs.d/microservice.tw/
sudo cp microservice.tw.cert /etc/docker/certs.d/microservice.tw/
sudo cp microservice.tw.key /etc/docker/certs.d/microservice.tw/
sudo cp ca.crt /etc/docker/certs.d/microservice.tw/
```

讓 Linux/Docker 接受自簽憑證 
---
```gherkin=
sudo cp microservice.tw.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
sudo systemctl restart docker.service
ls /etc/ssl/certs | awk /microservice.tw/
```

讓 Mac/Docker 接受自簽憑證 
---
```gherkin=
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain reg.microservice.tw.crt
```


harbor下載與設定
---
> 從 https://github.com/goharbor/harbor/releases 下載
```gherkin=
cp harbor.yml.template harbor.yml
```
> 修改
```gherkin=
hostname: reg.microservice.tw
certificate: /data/cert/microservice.tw.crt
private_key: /data/cert/microservice.tw.key
harbor_admin_password: VMware1!
```
> 執行 ./prepare & ./install

> 登入建好的 Harbor，驗證正確性
```gherkin=
docker login reg.microservice.tw
```
讓 minikube 接受自簽憑證 
---
```gherkin=
minikube start --insecure-registry="reg.microservice.tw"
```

> 驗證
> 先在 Harbor 建立一個repo
```gherkin=
docker pull nginx
docker tag nginx reg.microservice.tw/repo/nginx
docker push reg.microservice.tw/repo/nginx
```
```gherkin=
kubectl create deploy nginx --image=reg.microservice.tw/repo/nginx
kubectl get deploy nginx
```

讓 kapp-controller 接受自簽憑證(自行安裝 kapp-controller 為例)
---
> 修改
```gherkin=
k edit cm kapp-controller-config -n kapp-controller
```
> 將 microservice.tw.crt 內容寫入caCerts段，範例如下 
```gherkin=
apiVersion: v1
data:
  caCerts: |
    -----BEGIN CERTIFICATE-----
    MIIDWzCCAkOgAwIBAgIRAMODWpLzIWy3JocU4JBtIrIwDQYJKoZIhvcNAQELBQAw
    LTEXMBUGA1UEChMOUHJvamVjdCBIYXJib3IxEjAQBgNVBAMTCUhhcmJvciBDQTAe
    Fw0yMjAxMTExMDEwMDlaFw0zMjAxMDkxMDEwMDlaMC0xFzAVBgNVBAoTDlByb2pl
    Y3QgSGFyYm9yMRIwEAYDVQQDEwlIYXJib3IgQ0EwggEiMA0GCSqGSIb3DQEBAQUA

    [...]
    -----END CERTIFICATE-----    
  dangerousSkipTLSVerify: ""
  httpProxy: ""
  httpsProxy: ""
  noProxy: ""
``` 
> 刪除 kapp-controller-xxx pod，讓系統重新生成
```gherkin=
k delete pod kapp-controller-xxxxxx -n kapp-controller 
```

```gherkin=
export INSTALL_REGISTRY_USERNAME=admin
export INSTALL_REGISTRY_PASSWORD=VMware1!
export INSTALL_REGISTRY_HOSTNAME=harbor.microservice.tw
export TAP_VERSION=1.3.3
export INSTALL_REPO=tap133
```

```gherkin=
docker login registry.tanzu.vmware.com 

imgpkg tag list -i registry.tanzu.vmware.com/tanzu-application-platform/tap-packages | grep -v sha | sort -V

docker login $INSTALL_REGISTRY_HOSTNAME

imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages
```


```gherkin=
minikube start --kubernetes-version='1.22.8' --cpus='10' --memory='60g' --insecure-registry="harbor.microservice.tw"

minikube tunnel
```





參考
---
```gherkin=
https://rguske.github.io/post/deploy-tanzu-packages-from-a-private-registry/

https://itq.eu/knowledge/trusting-harbors-corporate-ca-in-tanzu-kubernetes-grid-with-carvel-tools/
```
