# アーキテクチャ概要

## 全体構成

本ハンズオンでは、Bastion を踏み台として Private EC2 に接続し、Private EC2 から FSx for NetApp ONTAP を NFS マウントする構成を採用しています。

Local PC
  ↓ SSH
Bastion EC2 (Public Subnet)
  ↓ SSH
Private EC2 (Private Subnet)
  ↓ NFS (TCP 2049)
FSx for NetApp ONTAP

この構成により、管理用アクセスとアプリケーション用アクセスの経路を分離しつつ、Private Subnet 上の EC2 からストレージへ安全に接続する流れを確認できます。

補足
本ハンズオンでは、接続経路を理解しやすくするために Bastion を利用した構成を採用しています。
実務では AWS Systems Manager Session Manager を利用して踏み台サーバを省略する構成もありますが、学習用としては接続経路が見えにくくなることや、追加構成が必要になることを考慮し、本ハンズオンでは採用していません。

## VPC構成

AWS 上の構成は以下の通りです。

VPC
├ Public Subnet AZ-a
│  ├ Bastion EC2
│  └ NAT Gateway
│
├ Private Subnet AZ-a
│  ├ Private EC2
│  └ FSx for NetApp ONTAP
│
└ Private Subnet AZ-c
   └ FSx for NetApp ONTAP

### Public Subnet

Public Subnet には、外部から SSH 接続を受け付ける Bastion EC2 と、Private Subnet からのアウトバウンド通信を中継する NAT Gateway を配置しています。

### Private Subnet

Private Subnet には、FSx for NetApp ONTAP をマウントする Private EC2 と、共有ストレージとして利用する FSx for NetApp ONTAP を配置しています。
FSx for NetApp ONTAP は Multi-AZ 構成を前提としているため、異なる Availability Zone にまたがる 2 つの Private Subnet を利用します。
これにより、単一 AZ 障害時の可用性を考慮した構成を学習できます。
Private EC2 は Public IP を持たず、インターネットから直接アクセスさせない構成としています。
管理用アクセスは Bastion を経由して行います。

## 接続経路

本ハンズオンで主に確認する接続経路は以下の通りです。

1. Local PC → Bastion EC2

ローカル PC から Bastion EC2 に対して SSH 接続を行います。
Bastion は Public Subnet 上に配置されており、管理用アクセスの入口となります。

Local PC
  ↓ SSH (TCP 22)
Bastion EC2

2. Bastion EC2 → Private EC2

Bastion から Private EC2 に対して SSH 接続を行います。
Private EC2 は Public IP を持たないため、直接アクセスではなく踏み台接続を利用します。

Bastion EC2
  ↓ SSH (TCP 22)
Private EC2

3. Private EC2 → FSx for NetApp ONTAP

Private EC2 から FSx for NetApp ONTAP の SVM endpoint に対して NFS 通信を行い、ボリュームをマウントします。

Private EC2
  ↓ NFS (TCP 2049)
FSx for NetApp ONTAP

4. Private EC2 → Internet

Private EC2 が OS パッケージの取得などで外部に通信する場合は、NAT Gateway を経由してインターネットへ接続します。

Private EC2
  ↓
NAT Gateway
  ↓
Internet

## 通信経路

本構成で利用する主な通信は以下の通りです。

| 通信元         | 通信先                  | 用途             | プロトコル / ポート    |
| ----------- | -------------------- | -------------- | -------------- |
| Local PC    | Bastion EC2          | 管理用 SSH 接続     | TCP 22         |
| Bastion EC2 | Private EC2          | 踏み台経由の SSH 接続  | TCP 22         |
| Private EC2 | FSx for NetApp ONTAP | NFS マウント       | TCP 2049       |
| Private EC2 | Internet             | パッケージ取得などの外部通信 | NAT Gateway 経由 |

未経験エンジニア向けには、単に接続できることだけでなく、「どの通信が何のためのものか」を意識して確認することが重要です。

## Multi-AZ 構成を採用している理由

本ハンズオンでは、FSx for NetApp ONTAP を Multi-AZ 構成とすることで、学習用の最小構成よりも実務に近い前提を取り入れています。

Single-AZ 構成は学習しやすい一方で、実運用を考えると可用性の面で制約があります。
一方、Multi-AZ 構成では異なる Availability Zone を利用することで、障害時の継続性を考慮した設計が可能になります。

本ハンズオンでは、以下のような観点を学ぶことを目的として Multi-AZ 構成を採用しています。

- なぜ Private Subnet が複数必要になるのか
- 学習用構成と本番想定構成で何が違うのか
- 可用性を意識したストレージ設計とは何か

## この構成で確認できること

本ハンズオンの構成では、以下のような実務でよくある確認ポイントを一通り体験できます。

- Bastion を踏み台とした Private EC2 への接続
- Private Subnet 上の EC2 から FSx for NetApp ONTAP への疎通確認
- NFS マウントの実施
- セキュリティグループやルート設定を意識した接続確認
- 権限エラー発生時の切り分け
- Single-AZ と Multi-AZ の違いの理解

このように、単なるサービス学習ではなく、詳細設計・実装・検証フェーズで担当しやすい作業の流れを意識した構成としています。

## CloudFormationスタックとの対応

本ハンズオンでは、環境を以下の 3 つの CloudFormation スタックに分割して管理しています。

- network
- compute
- storage

それぞれの役割は以下の通りです。

### network

- VPC
- Public Subnet
- Private Subnet (AZ-a / AZ-c)
- Internet Gateway
- Route Table
- Elastic IP
- NAT Gateway

### compute

- Bastion EC2
- Private EC2

### storage

- FSx for NetApp ONTAP

このようにスタックを分割することで、ネットワーク、コンピュート、ストレージを役割ごとに整理しやすくなり、構築・修正・削除時の管理もしやすくなります。

また、削除時は依存関係を考慮して以下の順で削除します。

storage
↓
compute
↓
network

## まとめ

本ハンズオンのアーキテクチャは、未経験エンジニアが実案件の中で担当しやすい詳細設計〜実装〜検証の流れを体験しやすいよう、必要な要素に絞って設計しています。

その上で、以下の実務に近いポイントを確認できる構成としています。

- Bastion を利用した管理アクセス
- Private Subnet 内の EC2 と FSx for NetApp ONTAP の接続
- Multi-AZ を前提としたストレージ構成
- NFS による共有ストレージの利用
- CloudFormation による構成管理
- 接続不具合や権限エラーの切り分け
