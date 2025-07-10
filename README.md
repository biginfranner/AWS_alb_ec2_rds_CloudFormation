
## 構成概要

* **VPC**：10.0.0.0/16
* **サブネット構成**（AZ a / c で以下を作成）

  * Public（ALB用）
  * Private Web（EC2用）
  * Private DB（RDS用）
  * Privateサブネットは各AZに1つ配置
* **IGWあり、NAT Gatewayあり**
* **AutoScaling構成**
* **EC2からSSMで接続**
* **Apacheインストール & RDS接続までUserDataで自動化**
---

---

###   VPC・サブネット

| 項目              | 値例                  |
| --------------- | ------------------- |
| VPC             | `10.0.0.0/16` |
| Public Subnet A | `10.0.0.0/24`       |
| Public Subnet B | `10.0.1.0/24`       |
| Web Subnet A    | `10.0.10.0/24`      |
| Web Subnet B    | `10.0.11.0/24`      |
| DB Subnet A     | `10.0.20.0/24`      |
| DB Subnet B     | `10.0.21.0/24`      |

---

###   ルートテーブル

| ルートテーブル  | 関連付けサブネット         | 設定     |
| -------- | ----------------- | ----------------- |
| PublicRT | Public Subnet A/B | `0.0.0.0/0 → IGW`     |
| PublicRT | Public Subnet A/B | `10.0.0.0/16 → local` |
| PrivateRT| Private Subnet A/B|  `0.0.0.0/0 → NGW`    |
| PrivateRT| Private Subnet A/B| `10.0.0.0/16 → local` |
---
###  　セキュリティグループ

#### インバウンド
| SG名    | 許可内容                  | 備考        |
| ------ | --------------------- | --------- |
| ALB-SG | TCP 80 from 0.0.0.0/0 | 公開        |
| EC2-SG | TCP 80 from ALB-SG    | ALB経由のみ許可 |
| RDS-SG | TCP 3306 from EC2-SG  | DB接続用     |

#### アウトバウンド(デフォルト)
| SG名    | 許可内容                  | 備考        |
| ------ | --------------------- | --------- |
| 全てのSG | すべてのトラフィック 80 from 0.0.0.0/0 | 公開        |

---

### EC2

* **AMI**：Amazon Linux  2023
* **インスタンスタイプ**：t3.micro
* **Subnet**：Private Subnet A と B
* **パブリックIPなし**
* **IAMロール**：`AmazonSSMManagedInstanceCore` をアタッチ
* **UserData**：

```bash
#!/bin/bash
dnf update -y 
dnf install -y httpd mysql
systemctl start httpd
systemctl enable httpd
echo "Hello from $(hostname)" > /var/www/html/index.html
```

---

###   RDS

* **エンジン**：MySQL
* **マルチAZ**：任意
* **DBサブネットグループ**：Private Subnet A/B を指定
* **セキュリティグループ**：RDS-SG を適用
* **パブリックアクセス**：**なし**

---

###   ALB（パブリックサブネット）

* **ターゲットグループ**：EC2インスタンスを登録（ポート80）
* **ALBのSG**：ALB-SG
* **サブネット**：Public Subnet A/B
* **リスナー**：HTTP（ポート80） → ターゲットグループ

---
