
## 構成概要

* **VPC**：10.0.0.0/16
* **サブネット構成**（AZ a / c で以下を作成）

  * Public（ALB用）
  * Private Web（EC2・DB用）
  * Privateサブネットは各AZに1つ配置
* **IGWあり、NAT Gatewayあり**
* **AutoScaling構成**
* **EC2からSSMで接続**
* **Apacheインストール & RDS接続までUserDataで自動化**
---


###   VPC・サブネット

| 項目              | 値例                  |
| --------------- | ------------------- |
| VPC             | `10.0.0.0/16` |
| Public Subnet A | `10.0.1.0/24`       |
| Public Subnet C | `10.0.2.0/24`       |
| Private Subnet A    | `10.0.10.0/24`      |
| Private Subnet C    | `10.0.11.0/24`      |

---

###   ルートテーブル

| ルートテーブル  | 関連付けサブネット         | 設定     |
| -------- | ----------------- | ----------------- |
| PublicRT | Public Subnet A/C | `0.0.0.0/0 → IGW`     |
| PublicRT | Public Subnet A/C | `10.0.0.0/16 → local` |
| PrivateRT| Private Subnet A/C|  `0.0.0.0/0 → NGW`    |
| PrivateRT| Private Subnet A/C| `10.0.0.0/16 → local` |
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
### EC2（AutoScaling管理）

| 項目               | 内容 |
|--------------------|------|
| **起動方式**        | AutoScaling（Launch Template）で2台自動起動 |
| **AMI**            | Amazon Linux 2023（SSM Parameter使用で常に最新） |
| **インスタンスタイプ** | t3.micro |
| **Subnet**         | Private Subnet A / C |
| **パブリックIP**    | なし |
| **IAMロール**       | `AmazonSSMManagedInstanceCore` をアタッチ |
| **UserData**       | Apacheインストール + ALB連携 + RDS接続（自動） |
| **ALB登録**        | ターゲットグループに自動登録 |
| **SSM接続**        | Session Manager で可能 |
| **スケール数**      | Min: 2, Max: 2（固定台数構成） |
| **ヘルスチェック**  | ALB経由で実施（異常時は自動再作成） |

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

###   ALB（パブリックサブネット）

* **ターゲットグループ**：EC2インスタンスを登録（ポート80）
* **ALBのSG**：ALB-SG
* **サブネット**：Public Subnet A/C
* **リスナー**：HTTP（ポート80） → ターゲットグループ

---
###   RDS

* **エンジン**：MySQL
* **マルチAZ**：任意
* **DBサブネットグループ**：Private Subnet A/C を指定
* **セキュリティグループ**：RDS-SG を適用
* **パブリックアクセス**：**なし**

---

