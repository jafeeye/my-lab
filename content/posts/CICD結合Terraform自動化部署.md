---
title: CICD結合Terraform自動化部署
toc: true
date: 2026-01-20
showTableOfContents: "true"
---
## **前言**
在現代軟體開發環境中，隨著系統規模與部署頻率不斷提升，傳統以人工方式進行程式建置與系統部署的作法，已逐漸無法滿足穩定性與效率的需求。人工流程不僅耗時，亦容易因人為操作失誤導致環境不一致、部署失敗或系統風險增加。
本專案以「自動化部署流程」為核心目標，透過整合版本控制、持續整合工具與基礎架構管理技術，建立一條從程式碼提交、建置、檢查到環境部署的自動化流程。藉由此流程，可減少人工介入，提升部署效率，並確保每次部署流程具備一致性與可追蹤性。
## **專案現況**
在專案導入自動化流程之前，系統部署與管理主要面臨以下現況與限制：
1.部署流程高度依賴人工操作
系統建置與部署流程多由人員手動執行，包含映像檔建置、上傳及環境部署。此方式不僅耗費人力，亦容易因操作不一致導致環境差異。
2.缺乏統一且可追蹤的部署流程
部署步驟未集中管理於版本控制系統中，當系統發生問題時，難以回溯部署來源與變更內容。
3.程式與映像檔建置流程未標準化
映像檔建置與推送流程缺乏固定規範，管理者無法快速確認映像檔是否來自合法、正確的建置流程。

## **解決方案**
為改善上述問題，本專案設計並實作一套自動化建置與部署流程，整合 GitLab CI/CD、容器技術與基礎架構即程式碼（Infrastructure as Code）概念，建立標準化的自動化 Pipeline，其流程如下：

1. 程式碼集中管理與自動觸發流程：開發人員將程式碼提交至 GitLab 後，系統即使用內建Gitlab Runner自動觸發 CI/CD Pipeline，確保每次變更皆經過相同的處理流程。
2. 自動化建置與檢查機制：Pipeline 中整合自動化檢查工具，使用GitLab中內建模板中SAST.gitlab-ci.yml 檢查原始碼漏洞、Dependency-Scanning.gitlab-ci.yml 檢查第三方套件相依性漏洞、Secret-Detection.gitlab-ci.yml 檢查程式碼的敏感憑證跟密鑰避免外洩。
3. 容器映像檔建置與掃描：系統自動建置 Docker 映像檔，將推送至私有映像檔倉庫（Harbor），使用GitLab中內建模板中Container-Scanning.gitlab-ci.yml的Trivy掃描部屬後的映像檔系統環境跟套件是否安全。
4. 映像檔簽署：使用Cosign簽署映像檔確保來源可被驗證，提升部署可信度。
5. 基礎架構自動化部署：將相關基礎架構設定程式化，透過Terraform + Cloud-init自動部署到地端 vSphere 中Rocky Linux虛擬機器，自動設定完網路與硬碟，安裝完Docker並從Harbor下載部屬完映像檔，使環境可重複建置且易於維護。

![](static/Pasted%20image%2020260927163310.png)

## 零、**部屬環境**

| **名稱**     | **系統**         | **IP**                | **用途**               |
| ---------- | -------------- | --------------------- | -------------------- |
| Runner VM  | Rocky Linux 10 | **192.168.80.102/24** | 設定自動化環境套件            |
| Gitlab VM  | Rocky Linux 10 | **192.168.80.107/24** | 地端託管程式碼              |
| Harbor VM  | Rocky Linux 10 | **192.168.80.108/24** | Docker Registry 倉庫服務 |
| vSphere VM | ESXi 8         | **192.168.80.110/24** | 地端環境                 |
| BindDNS VM | Rocky Linux 10 | **192.168.80.111/24** | DNS解析域名              |

## **一、建立Runner VM**

**安裝 Terraform、Git、Gitlab Runner、Docker**
```
#安裝Terraform

yum install -y yum-utils
yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
yum -y install terraform

#安裝Git
dnf -y install git

#安裝Gitlab Runner

curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | bash
dnf install -y gitlab-runner
systemctl enable --now gitlab-runner
systemctl status gitlab-runner --no-pager

#安裝 Docker

dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
ls -l /etc/yum.repos.d/
dnf install docker-ce -y
systemctl start docker
systemctl enable docker
systemctl status docker
```

## **二、建立 GitLab VM**
```
#安裝GitLab
sudo systemctl enable --now sshd
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo systemctl reload firewalld
sudo dnf install -y curl

curl "https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh" | sudo bash
sudo EXTERNAL\_URL="https://gitlab.x12lab.com" dnf install gitlab-ce
```

## **三、建立 Harbor VM**
```
#安裝Docker

dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

ls -l /etc/yum.repos.d/
dnf install docker-ce -y
systemctl start docker
systemctl enable docker
systemctl status docker

#安裝 Harbor Script
###################################################################

## 下載docker-compose

curl -L https://github.com/docker/compose/releases/download/v2.38.1/docker-compose-linux-x86\_64 > /usr/local/bin/docker-compose
sha256sum /usr/local/bin/docker-compose
chmod 755 /usr/local/bin/docker-compose
/usr/local/bin/docker-compose –version

## 下載harbor離線安裝包

cd /root
curl -LO https://github.com/goharbor/harbor/releases/download/v2.13.1/harbor-offline-installer-v2.13.1.tgz
tar zxvf harbor-offline-installer-v2.13.1.tgz

## 憑證相關
mkdir -p /etc/pki/tls/harbor
cd /etc/pki/tls/harbor

## 產生 CA private key

openssl genrsa -out ca.key 4096

## 產生 CA public key
openssl req -x509 -new -nodes -sha512 -days 3650 \
-subj "/C=TW/ST=Taiwan/L=Taipei/O=EXAMPLE/OU=DKL/CN=harbor.x12lab.com" \
-key ca.key \
-out ca.crt

## 產生 harbor.x12lab.com private key
openssl genrsa -out harbor.x12lab.com.key 4096

## 產生 harbor.x12lab.com CSR
openssl req -sha512 -new \
-subj "/C=TW/ST=Taiwan/L=Taipei/O=EXAMPLE/OU=DKL/CN=harbor.x12lab.com" \
-key harbor.x12lab.com.key \
-out harbor.x12lab.com.csr

## 使用CA private幫CRS sign
openssl x509 -req -sha512 -days 3650 \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-in harbor.x12lab.com.csr \
-out harbor.x12lab.com.crt

## docker 需要使用 .cert 副檔名
cp harbor.x12lab.com.crt harbor.x12lab.com.cert

## 部署 CA public key for docker
mkdir -p /etc/docker/certs.d/harbor.x12lab.com
cp /etc/pki/tls/harbor/ca.crt /etc/docker/certs.d/harbor.x12lab.com/.
cp /etc/pki/tls/harbor/harbor.x12lab.com.\* /etc/docker/certs.d/harbor.x12lab.com/.
systemctl restart docker

## 客製化的 harbor.yml
cd /root/harbor/
cp harbor.yml.tmpl harbor.yml
cat > harbor.sed <<-EOF
s/^hostname.\*/hostname\: harbor.x12lab.com/g
s/certificate\: \/your\/certificate\/path/certificate\: \/etc\/pki\/tls\/harbor\/harbor.x12lab.com.crt/g
s/private\_key\: \/your\/private\/key\/path/private\_key\: \/etc\/pki\/tls\/harbor\/harbor.x12lab.com.key/g
EOF
sed -i -f harbor.sed /root/harbor/harbor.yml

## 安裝harbor
cd /root/harbor
./install.sh
docker login -u admin -p Harbor12345 https://harbor.x12lab.com
```

## **四、建立 DNS VM**
```
## 安裝BIND DNS

dnf clean all
dnf makecache
dnf update -y openssl openssl-libs
dnf -y install bind-chroot
systemctl enable named-chroot
systemctl start named-chroot
systemctl status named-chroot

##建立named.conf 組態檔
vi /etc/named.conf
# listen-on port 53 { 127.0.0.1; };
# listen-on-v6 port 53 { ::1; };
allow-query { localhost;192.168.80.0/24;};
zone "x12lab.com" {
type master;
file "/var/named/x12lab.com.hosts";
allow-transfer{
192.168.80.71;
192.168.80.61;
};

notify yes;
};

zone "80.168.192.in-addr.arpa" {
type master;
file "/var/named/192.168.80.rev";
allow-transfer{
192.168.80.71;
192.168.80.61;
};

notify yes;
};

##建立x12lab.com.hosts DNS正向解析檔
vi /var/named/x12lab.com.hosts
$TTL 1D
@ IN SOA dns.x12lab.com. admin.x12lab.com. (
2025122601 ; Serial
1D ; Refresh
1H ; Retry
1W ; Expire
3H ) ; Minimum

; =========================
; Name Server
; =========================
@ IN NS dns.x12lab.com.
dns IN A 192.168.80.111

; =========================
; DevOps Services
; =========================
gitlab IN A 192.168.80.107
harbor IN A 192.168.80.108

; =========================
; vSphere / Virtualization
; =========================
esxi01 IN A 192.168.80.109
vcenter IN A 192.168.80.110

; =========================
; Future expansion (optional)
; =========================
terraform IN A 192.168.80.130
runner IN A 192.168.80.140

##建立192.168.80.rev DNS反向解析檔

vi /var/named/192.168.80.rev

$TTL 1D
@ IN SOA dns.x12lab.com. admin.x12lab.com. (
2025122601 ; Serial（有改就要+1）
1D ; Refresh
1H ; Retry
1W ; Expire
3H ) ; Minimum
@ IN NS dns.x12lab.com.
111 IN PTR dns.x12lab.com.
107 IN PTR gitlab.x12lab.com.
108 IN PTR harbor.x12lab.com.
110 IN PTR vcenter.x12lab.com.
109 IN PTR esxi01.x12lab.com.

## 重啟BindDNS並設定防火牆

named-checkconf /etc/named.conf
named-checkzone example.com /var/named/example.com.hosts
named-checkzone 80.168.192.in-addr.arpa /var/named/192.168.80.rev
systemctl restart named-chroot
systemctl status named-chroot
firewall-cmd --add-port=53/tcp --permanent
firewall-cmd --add-port=53/udp --permanent
firewall-cmd --reload

```

## **五、設定 Gitlab Runner Tag**
```
#設定憑證
echo | openssl s_client -connect gitlab.x12lab.com:443 -servername gitlab.x12lab.com 2>/dev/null \
| openssl x509 -outform PEM > /root/gitlab-ca.crt

#設定gitlab-runner
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.x12lab.com/" \
  --registration-token "打上真實tokenAPI" \
  --executor "shell" \
  --description "demo-runner" \
  --tag-list "demo" \
  --tls-ca-file "/root/gitlab-ca.crt"

#設定gitlab-runner
sudo gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.x12lab.com/" \
  --registration-token "打上真實TokenAPI" \
  --executor "docker" \
  --description "docker-runner" \
  --tag-list "docker" \
  --docker-image "docker:24.0" \
  --tls-ca-file "/root/gitlab-ca.crt"

##設定執行權限
  usermod -aG docker gitlab-runner
  getent group docker
  systemctl restart gitlab-runner
  sudo -u gitlab-runner docker ps

```

![](static/images/Pasted%20image%2020260927163733.png)

## **六、設定憑證**

**Runner VM Gitlab** 網頁https互相信任憑證
```
vi gitlab.conf
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
CN = gitlab.x12lab.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = gitlab.x12lab.com
IP.1  = 192.168.80.107
```

Runner VM 與Harbor互相信任憑證
├─ harbor-openssl.cnf
├─ harbor.key
├─ harbor.csr

製作harbor-openssl.cnf
```
harbor-openssl.cnf
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext
[ dn ]
CN = harbor.x12lab.com
[ req_ext ]
subjectAltName = @alt_names
[ alt_names ]
DNS.1 = harbor.x12lab.com
IP.1  = 192.168.80.20
```

製作Harbor VM的key + csr
```
openssl req -new -nodes -newkey rsa:2048 \
  -keyout harbor.key \
  -out harbor.csr \
  -config harbor-openssl.cnf

mkdir -p /root/harbor-ca
cd /root/harbor-ca
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key \
  -sha256 -days 3650 \
  -out ca.crt \
  -subj "/C=TW/O=x12lab/CN=x12lab-CA"
```
匯入OpenSSL憑證
```
openssl x509 -req \
  -in harbor.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out harbor.crt \
  -days 825 -sha256 \
  -extensions req_ext \
  -extfile gitlab.cnf
```

## **七、設定Gitlab Variables**

| 變數                  | 值                        |
| ------------------- | ------------------------ |
| HARBOR\_REGISTRY    | harbor.x12lab.com        |
| HARBOR\_USER        | harborops                |
| HARBOR\_PASS        | 2Passwo                  |
| IMAGE\_NAME         | library/hello            |
| VSPHERE\_SERVER     | vcenter.x12lab.com       |
| VSPHERE\_USER       | administrator@x12lab.com |
| VSPHERE\_PASSWORD   | 2Password!               |
| VSPHERE\_DATACENTER | vcdevops                 |
| VSPHERE\_HOST       | esxi01.x12lab.com        |
| VSPHERE\_DATASTORE  | datastore1               |
| VSPHERE\_NETWORK    | VMNetwork                |
| VSPHERE\_TEMPLATE   | rockylinux               |
| VM\_NAME            | rocky                    |
| VM\_CPU             | 2                        |
| VM\_MEM\_MB         | 4096                     |
| VM\_DISK\_GB        | 50                       |
| VM\_IP              | 192.168.80.60            |
| VM\_NETMASK         | 24                       |
| VM\_GW              | 192.168.80.2             |
| VM\_DNS             | 8.8.8.8                  |
| VM\_SSH\_USER       | root                     |
| VM\_SSH\_PASS       | 2Passwo                  |
|                     |                          |
![](static/images/Pasted%20image%2020260927163948.png)

## **八、撰寫文件檔**
在Gitlab Project名稱設定為cicd-demo，並撰寫相關程式碼，當開發人員push程式碼後便觸發.gitlab-ci.yml，進行test、build、scan、sign、deploy 所有步驟、最終結果可在瀏覽器輸入IP看見撰寫網頁內容。

**專案結構:**
```
cicd-demo/
 ├── Dockerfile：打包docker 映像
 ├── index.html：撰寫網頁程式碼
 ├── .gitlab-ci.yml：GitLab CI/CD 的自動化腳本
 ├── README.md
 └── infra/
   ├── cloud-init.yaml：設定使用者名稱、自動安裝憑證、Docker、拉取Harbor映像檔
   ├── main.tf：讀取cloud-init.yaml
   ├── network-config.yaml：設定網路環境
   ├── variables.tf：讀取GitLab Variables 與先設定好變數
   └── outputs.tf： 傳遞SSH的vm_ip
```

![](static/images/Pasted%20image%2020260927164014.png)
### index.html
```
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>歡迎 - DevOps Demo</title>
    <style>
        /* 初始化設定 */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Noto Sans TC', 'Helvetica Neue', Arial, sans-serif;
            /* 漂亮的漸層背景 */
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            color: #333;
            padding: 20px;
        }

        /* 主要內容卡片區塊 */
        .welcome-card {
            background: rgba(255, 255, 255, 0.95);
            padding: 3rem;
            border-radius: 20px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.2);
            text-align: center;
            max-width: 500px;
            width: 100%;
            /* 簡單的進場動畫 */
            animation: fadeInUp 0.8s ease-out;
        }

        /* 標題樣式 */
        .welcome-card h1 {
            font-size: 2.5rem;
            margin-bottom: 1rem;
            color: #4a4a4a;
            font-weight: 700;
        }
        
        /* 強調文字顏色 */
        .highlight {
            background: linear-gradient(to right, #667eea, #764ba2);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }

        /* 副標題文字樣式 */
        .welcome-card p {
            font-size: 1.1rem;
            line-height: 1.6;
            color: #666;
            margin-bottom: 2rem;
        }

        /* 裝飾用的按鈕 (目前沒有功能) */
        .btn {
            display: inline-block;
            padding: 12px 30px;
            background: linear-gradient(to right, #667eea, #764ba2);
            color: white;
            text-decoration: none;
            border-radius: 50px;
            font-weight: 600;
            transition: transform 0.3s ease, box-shadow 0.3s ease;
        }

        .btn:hover {
            transform: translateY(-3px);
            box-shadow: 0 5px 15px rgba(102, 126, 234, 0.4);
        }
        
        /* 頁尾小字 */
        .footer-text {
            margin-top: 2rem;
            font-size: 0.8rem;
            color: #999;
        }

        /* 動畫定義 */
        @keyframes fadeInUp {
            from {
                opacity: 0;
                transform: translateY(30px);
            }
            to {
                opacity: 1;
                transform: translateY(0);
            }
        }
    </style>
</head>
<body>
    <div class="welcome-card">
        <h1>歡迎來到 <span class="highlight">DevOps</span> Demo</h1>
        <p>
            您現在看到的頁面，是透過 GitLab CI/CD 自動化流水線成功構建並部署的靜態網站成果。
        </p>
        <a href="#" class="btn">探索更多</a>
        <div class="footer-text">
            Deployed via GitLab CI/CD & Terraform
        </div>
    </div>
</body>
</html>
```

### **Dockerfile**
```
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
RUN echo "Hello from GitLab CI $(date)" > /usr/share/nginx/html/index.html
```

### **cloud-init.yaml**
```
#cloud-config
hostname: ${VM_NAME}
manage_etc_hosts: true

users:
  - name: rocky
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: [docker]
    shell: /bin/bash
    lock_passwd: false

# demo 先用密碼，之後可改 ssh key
chpasswd:
  expire: false
  list: |
    rocky:${VM_SSH_PASS}

ssh_pwauth: true

package_update: true
packages:
  - ca-certificates
  - curl
  - open-vm-tools
  - NetworkManager
  # 觀測
  - wget
  - tar

write_files:
  - path: /etc/pki/ca-trust/source/anchors/harbor-ca.crt
    permissions: '0644'
    content: |
      -----BEGIN CERTIFICATE-----
      MIIFRTCCAy2gAwIBAgIUV8bhoHlOc+HQ4qgWJXT0ynxJ41YwDQYJKoZIhvcNAQEL
      BQAwMjELMAkGA1UEBhMCVFcxDzANBgNVBAoMBngxMmxhYjESMBAGA1UEAwwJeDEy
      bGFiLUNBMB4XDTI1MTIyNzA3Mzg1M1oXDTM1MTIyNTA3Mzg1M1owMjELMAkGA1UE
      BhMCVFcxDzANBgNVBAoMBngxMmxhYjESMBAGA1UEAwwJeDEybGFiLUNBMIICIjAN
      BgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAsVl5P6jv2Zz35i2uuWi4k8CZaIRu
      ym8BYE/nkLfP6VQ3C7bb/qEIVv48IViJlZkK9yn2kC7oSfLj3M1OIk7kqrNgnStQ
      HdSjqdmHHzLwqvb8gzOSpt1xRT1hXSjpbuZkfzi8XVpck4iQlL+Z5HaBfNSLgpif
      +D6E1IHy+CcOeT6xSAXDs1HOQELRWdlxYAaCaVyX8qE0JSN/OOFYy12P3uejI0hI
      TSxpMGa1b2W+5usAmRhIDkkZIMmE6U0EH5A1Tq1LOKoAur76pJL9N1rWryo/6NIS
      DobfFoP7q7AZSSlDxfElJ4zXwdJuFY4pjsqB2EKLtJfBZcxBEmBzZ28y3VQZjJYU
      T2BNnK2KzAhw1Iw3z7uJf16xncySyTVFiv4r3nbNZhbOGHalmGOyUZ6pqc0zb6fw
      Kdupy5eleOdlYIcsSHHmA+AstMnEQQ6yyN16zJPnobBfs0nkwPOPrmWPNv5UXNzn
      992t8AAipXSj4OMt5JFif0F9yPLCdd1t6WTIBR8QCGXrp/EuiNFhnaFJelG452Ry
      HcXIM5+r36xJ4Qme/1BZqN6k1czbJ3B284HlJgeCMVYpXKcOYzQBg05gF3sy9unM
      cLXJqyMhhyncClsgy139ND0U1uOhAFNe57CVr30P3P16tmOTlbatAKDpmlSsHfKh
      Q+WOReBBr1umYrMCAwEAAaNTMFEwHQYDVR0OBBYEFOfKkrbeeOQrfzuSzbqGO+hO
      Wex/MB8GA1UdIwQYMBaAFOfKkrbeeOQrfzuSzbqGO+hOWex/MA8GA1UdEwEB/wQF
      MAMBAf8wDQYJKoZIhvcNAQELBQADggIBAHJ7ndMFWix/0C55MeMcERt3kYXrajQd
      j9hrzbQQvx9YXaRN5yOeQCUkxesxT1HrMQ9VDsQNhQEYyDfJnBw0Mx6RR61INCGo
      e67/HlY7+XJVAGWxuAGMe9T1kFiiziqaN7SzuWFoEMwFywtYF4GQ2bGJ+k+IZtDN
      xcBP0oUA1YQats1OelsyuIzXqZgHO0MhRMeVK86yoV7VYH/fowXQbBJR0WxlUD5q
      qe5SeUge85qXrA8zMUenqV2RlNXG9V8hgcdzCgL1X13SLW4CodFBec7tZ2DeIm6q
      w4q9dBQePV08MLjqEfQMyMLyy7n38cRKYNU//ZQrrJTsaKB5feL9kSL1EMgfEXPn
      oS9It6uYBFM/sqZuIA0IUuWW0ZhDuK/QrIRQWEk9wNxwVZFab63UuvYcctzqYWTG
      Z5iVJDo3sgYwr5B43SPWqgGbGdi0XePDgBXIRPP6laZqDM5mrE/nZtWT8AApTwda
      CQQLom6VZGr7ZX36bIKwJcF/xTEP8duCpUwEH7jUff3o4IGepxhaBk+G2/p3uz7i
      MJRl5nDELmu86DhqtmG2l066IqW8HS1iK/uSy3N/S/p5x8aIZfGkEQA6FEeJHvtQ
      okGXg+twMh61p0a7W56zuV/KSau9KrLcSTum97asSAZ8jKc9IfLj958fB60r9D+v
      9RPV6W7q29IY
      -----END CERTIFICATE-----
 
runcmd:
  - systemctl enable --now vmtoolsd || true
  - systemctl enable --now NetworkManager || true
  - resolvectl status || true

  # trust harbor CA
  - update-ca-trust || true

  # install docker (Rocky)
  - dnf -y install dnf-plugins-core
  - dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
  - dnf -y install docker-ce docker-ce-cli containerd.io docker-compose-plugin
  - systemctl enable --now docker
  - systemctl restart docker || true
  - usermod -aG docker rocky || true

  # run app
  - docker login ${HARBOR_REGISTRY} -u "${HARBOR_USER}" -p "${HARBOR_PASS}"
  - docker pull ${APP_IMAGE}
  - docker rm -f web || true
  - docker run -d --name web -p 80:80 ${APP_IMAGE}


  # open ports (if firewalld exists)
  - firewall-cmd --permanent --add-port=3000/tcp --add-port=9090/tcp --add-port=9100/tcp || true
  - firewall-cmd --reload || true
```

###  .gitlab-ci.yml
```
stages:
  - test
  - build
  - scan
  - sign
  - deploy

include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml
  - template: Jobs/Container-Scanning.gitlab-ci.yml

variables:
  CS_IMAGE: "$HARBOR_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"

hello_ci:
  stage: test
  tags:
    - demo
  script:
    - echo "Hello GitLab CI"

container_scanning:
  stage: scan
  tags:
    - docker
  needs:
    - build_push


secret_detection:
  tags:
    - docker

sast:
  tags:
    - docker

dependency_scanning:
  tags:
    - docker

build_push:
  stage: build
  tags:
    - demo
  script:
    - docker login $HARBOR_REGISTRY -u "$HARBOR_USER" -p "$HARBOR_PASS"
    - docker build -t $HARBOR_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker push $HARBOR_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA

image_sign:
  stage: sign
  tags: ["docker"]
  image: alpine:3.20
  needs:
    - build_push
    - container_scanning
  variables:
    COSIGN_VERSION: "v2.4.3"
  script:
    - apk add --no-cache curl ca-certificates
    - curl -Lo /usr/local/bin/cosign "https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64"
    - chmod +x /usr/local/bin/cosign
    - cosign version
    # 還原私鑰
    - echo "$COSIGN_PRIVATE_KEY" | base64 -d > cosign.key
    # (可選但建議) 先登入 Harbor，避免 unauthorized
    - echo "$HARBOR_PASS" | cosign login "$HARBOR_REGISTRY" -u "$HARBOR_USER" --password-stdin
    # 簽署（Harbor）
    - cosign sign --key cosign.key --allow-insecure-registry "$HARBOR_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"

terraform_apply:
  stage: deploy
  tags: [demo]
  script:
    - cd infra
    - terraform init
    - terraform apply -auto-approve
      -var="vsphere_server=$VSPHERE_SERVER"
      -var="vsphere_user=$VSPHERE_USER"
      -var="vsphere_password=$VSPHERE_PASSWORD"
      -var="datacenter=$VSPHERE_DATACENTER"
      -var="host=$VSPHERE_HOST"
      -var="datastore=$VSPHERE_DATASTORE"
      -var="network=$VSPHERE_NETWORK"
      -var="template=$VSPHERE_TEMPLATE"
      -var="vm_name=$VM_NAME"
      -var="vm_cpu=$VM_CPU"
      -var="vm_mem_mb=$VM_MEM_MB"
      -var="vm_disk_gb=$VM_DISK_GB"
      -var="vm_ip=$VM_IP"
      -var="vm_netmask=$VM_NETMASK"
      -var="vm_gw=$VM_GW"
      -var="vm_dns=$VM_DNS"
      -var="ssh_user=$VM_SSH_USER"
      -var="ssh_pass=$VM_SSH_PASS"
      -var="harbor_registry=$HARBOR_REGISTRY"
      -var="harbor_user=$HARBOR_USER"
      -var="harbor_pass=$HARBOR_PASS"
      -var="app_image=$HARBOR_REGISTRY/$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"
```

###  Main.tf
```
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    vsphere = {
      source  = "hashicorp/vsphere"
      version = "~> 2.5"
    }
  }
}

provider "vsphere" {
  vsphere_server       = var.vsphere_server
  user                 = var.vsphere_user
  password             = var.vsphere_password
  allow_unverified_ssl = true
}

data "vsphere_datacenter" "dc" {
  name = var.datacenter
}

data "vsphere_host" "host" {
  name          = var.host
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_datastore" "ds" {
  name          = var.datastore
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_network" "net" {
  name          = var.network
  datacenter_id = data.vsphere_datacenter.dc.id
}

data "vsphere_virtual_machine" "tpl" {
  name          = var.template
  datacenter_id = data.vsphere_datacenter.dc.id
}

locals {
  user_data = templatefile("${path.module}/cloud-init.yaml", {
    VM_NAME         = var.vm_name
    VM_SSH_PASS     = var.ssh_pass
    HARBOR_REGISTRY = var.harbor_registry
    HARBOR_USER     = var.harbor_user
    HARBOR_PASS     = var.harbor_pass
    APP_IMAGE       = var.app_image
  })
  
  network_config = templatefile("${path.module}/network-config.yaml", {
    VM_IP      = var.vm_ip
    VM_NETMASK = var.vm_netmask
    VM_GW      = var.vm_gw
    VM_DNS     = var.vm_dns
  })

}

resource "vsphere_virtual_machine" "vm" {
  name             = var.vm_name
  resource_pool_id = data.vsphere_host.host.resource_pool_id
  datastore_id     = data.vsphere_datastore.ds.id

  num_cpus = var.vm_cpu
  memory   = var.vm_mem_mb
  guest_id = data.vsphere_virtual_machine.tpl.guest_id
  scsi_type = data.vsphere_virtual_machine.tpl.scsi_type
  
  firmware = "efi"
  efi_secure_boot_enabled = false

  extra_config = {
    "guestinfo.userdata"          = base64encode(local.user_data)
    "guestinfo.userdata.encoding" = "base64"
    "guestinfo.metadata"          = base64encode(local.network_config)
    "guestinfo.metadata.encoding" = "base64"
  }
  
  network_interface {
    network_id   = data.vsphere_network.net.id
    adapter_type = data.vsphere_virtual_machine.tpl.network_interface_types[0]
  }

  disk {
    label            = "disk0"
    size             = var.vm_disk_gb
    thin_provisioned = true
  }

  clone {
    template_uuid = data.vsphere_virtual_machine.tpl.id
  }

  # SSH 進 VM 安裝 docker + 跑 image

}
```

### network-config.yaml
```
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      dhcp6: false
      addresses:
        - ${VM_IP}/${VM_NETMASK}
      gateway4: ${VM_GW}
      nameservers:
        addresses:
          - 192.168.80.111
```

### outputs.tf
```
output "vm_ip" {
  value = var.vm_ip
}
```

### variables.tf
```
variable "vsphere_server" {}
variable "vsphere_user" {}
variable "vsphere_password" {}

variable "datacenter" {}
variable "host" {}
variable "datastore" {}
variable "network" {}
variable "template" {}

variable "vm_name" {}
variable "vm_cpu" { type = number }
variable "vm_mem_mb" { type = number }
variable "vm_disk_gb" { type = number }

variable "vm_ip" {}
variable "vm_netmask" { type = number }
variable "vm_gw" {}
variable "vm_dns" {}

variable "ssh_user" {}
variable "ssh_pass" {}

variable "harbor_registry" {}
variable "harbor_user" {}
variable "harbor_pass" {}
variable "app_image" {}
```

**Pipeline** **進行畫面**
![](static/images/Pasted%20image%2020260927164042.png)

![](static/images/Pasted%20image%2020260927164045.png)

## **九、結語**

本專案成功整合了 GitLab CI、Harbor 與 Terraform，展示了一套完整的現代化 IT 交付流程。我們利用 基礎設施即代碼 (IaC) 技術，將繁瑣的 VMware vSphere 虛擬機部署轉化為可版控、可重複執行的代碼。
從開發者提交網頁程式碼的那一刻起，系統便自動完成建置、資安檢核到基礎設施的佈建，完全消除了人為操作的失誤風險，為企業級的持續整合與持續部署 (CI/CD) 提供了標準化的解決方案。
