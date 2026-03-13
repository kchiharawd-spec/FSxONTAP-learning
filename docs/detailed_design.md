# 詳細設計

## 設計方針

本ハンズオンは、未経験エンジニアが実案件に参画した際に、詳細設計以降の工程をイメージしやすくすることを目的として設計している。  
そのため、単に FSx for NetApp ONTAP を利用する手順を学ぶのではなく、**詳細設計、実装、検証、運用を見据えた確認**の流れを意識した構成としている。

設計にあたっては、以下の方針を採用した。

### 1. 実務で扱いやすい構成を意識する
未経験エンジニアが現場で最初に担当しやすいのは、要件定義や基本設計よりも、詳細設計を確認しながらの構築、接続確認、検証、一次切り分けといった工程である。  
そのため本ハンズオンでは、実装終盤から検証フェーズで実際に触れやすい作業を再現しやすい構成を採用している。

### 2. 学習しやすさと実務らしさのバランスを取る
学習用の構成はできるだけシンプルであることが望ましいが、簡略化しすぎると実務との乖離が大きくなる。  
そのため、接続経路の理解を優先して Bastion を利用しつつ、ストレージについては実務を意識して FSx for NetApp ONTAP を Multi-AZ 構成とし、学習しやすさと実務らしさの両立を図っている。

### 3. 接続経路が理解しやすい構成とする
未経験エンジニアにとって、どこからどこへ、どの通信で接続しているのかを把握することは重要である。  
そのため、Local PC から Bastion、Bastion から Private EC2、Private EC2 から FSx for NetApp ONTAP という流れが明確に見える構成とし、接続確認やトラブルシューティングを行いやすくしている。

### 4. セキュリティを意識した配置とする
実務では、アプリケーションサーバやストレージをインターネットから直接アクセス可能な状態にしないことが基本となる。  
そのため、本ハンズオンでも Private EC2 および FSx for NetApp ONTAP は Private Subnet に配置し、管理用アクセスは Bastion 経由に限定する構成としている。

### 5. 可用性を考慮したストレージ構成とする
学習用の最小構成では Single-AZ の方が扱いやすいが、本番環境では可用性を考慮して Multi-AZ 構成を採用することが多い。  
そのため本ハンズオンでは、FSx for NetApp ONTAP を Multi-AZ 前提とし、異なる Availability Zone にまたがるサブネット構成を採用している。  
これにより、Single-AZ 構成との違いや、可用性を考慮した設計の考え方も学べるようにしている。

### 6. IaC による再現性を重視する
構成管理や再作成のしやすさは、実務でも重要な観点である。  
そのため、本ハンズオンでは CloudFormation を利用し、network、compute、storage の 3 スタックに分割して環境を管理している。  
これにより、役割ごとに構成を整理しやすくし、構築、修正、削除の流れも把握しやすくしている。

### 7. 検証と切り分けをしやすい構成とする
本ハンズオンでは、最終的に Private EC2 から FSx for NetApp ONTAP を NFS マウントし、ファイル作成まで確認することをゴールとしている。  
そのため、名前解決、SSH 接続、NFS ポート疎通、マウント、権限確認といった観点で段階的に確認できるよう構成している。  
これにより、問題発生時にどのレイヤで詰まっているのかを切り分けしやすくしている。

## ネットワーク設計

本ハンズオンでは、管理用アクセスと内部通信の役割を分離し、未経験エンジニアでも接続経路を理解しやすい構成とするために、VPC 内に Public Subnet と Private Subnet を分けて設計している。

### 1. VPC設計

VPC は、ハンズオン全体で利用するネットワーク基盤として作成する。  
EC2、NAT Gateway、FSx for NetApp ONTAP を同一 VPC 内に配置し、各コンポーネント間の通信を VPC 内で完結できるようにする。

想定する CIDR は以下の通りとする。

VPC CIDR: 10.0.0.0/16

## サブネット設計

本ハンズオンでは、1 つの Public Subnet と 2 つの Private Subnet を作成する。
Private Subnet を 2 つ用意する理由は、FSx for NetApp ONTAP を Multi-AZ 構成で利用するためである。

想定するサブネットは以下の通りとする。

- Public Subnet AZ-a   : 10.0.1.0/24
- Private Subnet AZ-a  : 10.0.11.0/24
- Private Subnet AZ-c  : 10.0.12.0/24

## EC2設計

本ハンズオンでは、EC2 は役割の異なる 2 台を配置する。

- Bastion EC2
- Private EC2

それぞれの EC2 は役割が明確に異なるため、配置先サブネットや接続方式も分けて設計する。

---

### 1. Bastion EC2

Bastion EC2 は、外部からの管理用アクセスを受け付ける踏み台サーバとして利用する。  
Private Subnet に配置した EC2 はインターネットから直接アクセスできないため、Bastion を経由して接続する構成とする。

#### 配置先
- Public Subnet AZ-a

#### 役割
- Local PC からの SSH 接続を受け付ける
- Private EC2 への踏み台として利用する

#### 設計意図
Bastion を配置することで、Private EC2 に Public IP を持たせずに管理アクセスを実現できる。  
これにより、業務用サーバをインターネットへ直接公開しない構成を学習できる。

また、未経験エンジニアにとっては、

- Local PC → Bastion
- Bastion → Private EC2

という接続経路が明確になるため、接続確認やトラブルシューティングの流れを理解しやすい。

---

### 2. Private EC2

Private EC2 は、FSx for NetApp ONTAP をマウントして動作確認を行う業務サーバを想定したインスタンスである。  
本ハンズオンでは、この EC2 から FSx for NetApp ONTAP の SVM endpoint に対して NFS マウントを実施する。

#### 配置先
- Private Subnet AZ-a

#### 役割
- FSx for NetApp ONTAP のマウント元となる
- NFS の疎通確認、マウント、ファイル作成確認を行う
- 検証用サーバとして利用する

#### 設計意図
Private EC2 は、実際の業務サーバや検証サーバをイメージした配置としている。  
Public IP は付与せず、Bastion 経由でのみアクセス可能とすることで、実務で一般的な Private 配置の考え方を学べるようにしている。

また、FSx for NetApp ONTAP のマウント確認やファイル書き込み確認を行うことで、ストレージ移行案件の実装終盤〜検証フェーズで実際に発生しやすい作業を再現している。

---

### 3. インスタンス配置の考え方

本ハンズオンでは、Bastion EC2 と Private EC2 を以下のように配置する。

Public Subnet AZ-a
└ Bastion EC2

Private Subnet AZ-a
└ Private EC2

### 4. 接続方式

EC2 間および外部からの接続は、以下の流れを前提とする。

Local PC
  ↓ SSH
Bastion EC2
  ↓ SSH
Private EC2

この構成により、未経験エンジニアでも接続経路を追いやすく、問題発生時にどの区間で失敗しているかを切り分けしやすい。

### 5. 外向き通信の考え方

Private EC2 はインターネットへ直接接続せず、必要な外向き通信のみ NAT Gateway 経由で行う。
例えば以下のような通信が該当する。

- OS パッケージ取得
- yum / dnf 実行
- パッケージインストール時のリポジトリアクセス

一方で、Private EC2 から FSx for NetApp ONTAP への通信は VPC 内通信であり、NAT Gateway は経由しない。

### 6. 学習用としての位置づけ

実務では、業務サーバが複数台存在することや、踏み台サーバを使わずに Session Manager などを利用することもある。
一方、本ハンズオンでは接続経路を理解しやすくすることを優先し、Bastion EC2 と Private EC2 の 2 台構成としている。

この構成により、未経験エンジニアでも以下を段階的に理解しやすくなる。

- Bastion の役割
- Private EC2 の役割
- Public / Private の配置の違い
- SSH 接続の流れ
- FSx for NetApp ONTAP を利用する検証サーバの位置づけ

そのため、本ハンズオンでは EC2 を最小限の構成にしつつ、実務でよくある考え方を体験できる形としている。

## FSx for NetApp ONTAP 設計

本ハンズオンでは、オンプレミス環境で利用している NetApp ストレージの移行先として、AWS の **FSx for NetApp ONTAP** を利用する。  
Private EC2 から NFS 経由でアクセスし、共有ストレージとして利用する構成を前提とする。

FSx for NetApp ONTAP は、単なるファイル共有ストレージとして扱うだけでなく、NetApp ONTAP の機能を AWS 上で利用できるマネージドサービスである。  
そのため本ハンズオンでは、AWS のストレージサービスとしての利用だけでなく、NetApp ストレージを AWS に移行する案件を意識した構成としている。

---

### 1. 採用理由

本ハンズオンで FSx for NetApp ONTAP を採用した理由は以下の通りである。

- オンプレミス NetApp から AWS への移行先として利用されるケースがある
- NFS を利用したファイル共有構成を再現できる
- 実務でよく意識される SVM や Volume といった ONTAP の概念を学べる
- 単なる AWS サービスの利用ではなく、移行案件を意識した学習ができる

本ハンズオンでは、FSx for NetApp ONTAP を「共有ストレージ」として利用しつつ、実案件で登場しやすい構成要素に触れることを目的としている。

---

### 2. 配置方針

FSx for NetApp ONTAP は、Private Subnet 内に配置する。  
また、本ハンズオンでは実務寄りの構成を意識し、**Multi-AZ 構成**を前提とする。

#### 配置先
- Private Subnet AZ-a
- Private Subnet AZ-c

#### 設計意図
FSx for NetApp ONTAP は共有ストレージであり、インターネットから直接アクセスさせるものではないため、Private Subnet に配置する。  
また、Single-AZ 構成ではなく Multi-AZ 構成を採用することで、学習用の最小構成よりも本番環境に近い可用性の考え方を取り入れている。

この構成により、以下を理解しやすくしている。

- FSx for NetApp ONTAP は Private に配置することが基本であること
- Multi-AZ 構成では複数のサブネットが必要になること
- 学習用構成と本番想定構成の違い

---

### 3. 利用方式

本ハンズオンでは、Private EC2 から FSx for NetApp ONTAP に対して **NFS 接続**を行う。  
接続先として利用するのは、FSx ファイルシステム本体のエンドポイントではなく、**SVM endpoint** である。

Private EC2
  ↓ NFS (TCP 2049)
SVM endpoint
  ↓
Volume

### 4. 構成要素

本ハンズオンでは、FSx for NetApp ONTAP の構成要素を以下の単位で整理する。

#### File System
AWS 上で作成する FSx for NetApp ONTAP のストレージ基盤。
本ハンズオンでは Multi-AZ 構成のファイルシステムを利用する。

#### Storage Virtual Machine (SVM)
クライアントがアクセスする論理的なストレージ管理単位。
NFS マウント時には、この SVM に紐づく endpoint を利用する。

#### Volume
実際にマウント対象となる領域。
本ハンズオンでは、Volume を NFS でマウントし、ファイル作成による動作確認を行う。

この 3 つの関係は以下のように整理できる。

FSx for NetApp ONTAP
└ Storage Virtual Machine (SVM)
   └ Volume

### 5.接続設計

Private EC2 から FSx for NetApp ONTAP への接続は、VPC 内通信として行う。
そのため、インターネット接続や NAT Gateway を経由せず、VPC 内の local ルートで通信する。

利用するプロトコルおよびポートは以下の通り。

- プロトコル: TCP
- ポート: 2049
- 用途: NFS マウント

### 6.マウント対象

本ハンズオンでは、Private EC2 から FSx for NetApp ONTAP の Volume を NFS マウントし、動作確認を行う。
マウント対象は、SVM endpoint に紐づく Volume とする。

例: 
<SVM endpoint>:/vol1

このマウントにより、以下を確認する。

- NFS クライアント設定が正しいこと
- SVM endpoint の名前解決ができること
- NFS ポートに疎通できること
- 実際にファイル作成ができること
- 権限エラー発生時に切り分けできること

### 7.学習上の確認ポイント

本ハンズオンでは、FSx for NetApp ONTAP を利用するうえで以下の点を確認できるようにしている。

- FSx for NetApp ONTAP は Private Subnet に配置する
- Multi-AZ 構成ではサブネットが複数必要になる
- NFS マウントには SVM endpoint を利用する
- FSx endpoint と SVM endpoint は異なる
- Volume の権限によって書き込み可否が変わる
- 通信確認、マウント確認、権限確認を段階的に実施する

### 8.学習用として簡略化している点

実務では、FSx for NetApp ONTAP に対して Snapshot、SnapMirror、監視設定、バックアップ方針など、さらに多くの設計観点が存在する。
一方、本ハンズオンでは未経験エンジニアが実装終盤〜検証フェーズで触れやすい範囲に絞り、以下を中心に扱う。

- SVM endpoint を利用した接続
- NFS マウント
- ファイル作成による動作確認
- 権限エラーの確認
- Multi-AZ を前提とした基本構成

このように、FSx for NetApp ONTAP の設計要素を絞ることで、ストレージ移行案件に必要な基礎理解を得やすくしている。

## ルート設計

本ハンズオンでは、Public Subnet と Private Subnet で役割が異なるため、それぞれに対応したルート設計を行う。  
ルート設計の目的は、**外部公開が必要な通信**と**内部通信のみで完結すべき通信**を明確に分けることである。

本構成では、以下の 2 種類の通信を意識して設計する。

- インターネットとの通信
- VPC 内部の通信

---

### 1.Public Subnet のルート設計

Public Subnet には Bastion EC2 と NAT Gateway を配置するため、インターネットと通信できる必要がある。  
そのため、Public Subnet に関連付けるルートテーブルには、デフォルトルートを Internet Gateway に向ける。

0.0.0.0/0 → Internet Gateway

#### 設計意図
この設定により、Public Subnet 上の Bastion EC2 はローカル PC からの SSH 接続を受け付けることができる。
また、NAT Gateway 自体も Public Subnet に配置されるため、外部との通信経路として Internet Gateway が必要になる。

### 2.Private Subnet のルート設計

Private Subnet には Private EC2 および FSx for NetApp ONTAP を配置する。
これらのリソースはインターネットから直接アクセスさせないため、Private Subnet に関連付けるルートテーブルでは、デフォルトルートを NAT Gateway に向ける。

0.0.0.0/0 → NAT Gateway

#### 設計意図
この設定により、Private EC2 は Public IP を持たなくても、必要な場合に限り外部へ通信できる。
例えば、以下のような通信で利用する。

- OS パッケージの取得
- yum / dnf によるパッケージインストール
- 外部リポジトリへのアクセス

一方で、インターネットから Private EC2 や FSx for NetApp ONTAP へ直接アクセスすることはできない。

### 3.VPC内部通信の考え方

VPC 内には自動的に local ルートが存在する。
このルートにより、同一 VPC 内の CIDR に属する通信は VPC 内部で完結する。

10.0.0.0/16 → local

#### 設計意図
この local ルートにより、以下のような通信は NAT Gateway や Internet Gateway を経由しない。

- Bastion EC2 → Private EC2
- Private EC2 → FSx for NetApp ONTAP
- EC2 間の内部通信

特に本ハンズオンでは、Private EC2 から FSx for NetApp ONTAP への NFS 通信は VPC 内部通信であることを理解することが重要である。
この通信はインターネット向け通信ではないため、NAT Gateway の有無とは別に考える必要がある。

### 4.通信経路ごとのルートの違い

本ハンズオンで扱う主要な通信と、利用されるルートの関係は以下の通りである。

| 通信                                 | 利用ルート            | 備考                           |
| ---------------------------------- | ---------------- | ---------------------------- |
| Local PC → Bastion EC2             | Internet Gateway | Bastion は Public Subnet 上に配置 |
| Bastion EC2 → Private EC2          | local            | VPC 内部通信                     |
| Private EC2 → FSx for NetApp ONTAP | local            | VPC 内部通信                     |
| Private EC2 → Internet             | NAT Gateway      | OS パッケージ取得など                 |

このように、同じ Private EC2 からの通信であっても、接続先によって利用する経路が異なる点が重要である。

### 5.Multi-AZ 構成におけるルート設計

本ハンズオンでは FSx for NetApp ONTAP を Multi-AZ 構成とするため、Private Subnet は AZ-a と AZ-c の 2 つを利用する。
ただし、ルート設計の基本的な考え方は両方の Private Subnet で同じであり、どちらもデフォルトルートは NAT Gateway に向ける。

例: 
Private Subnet AZ-a
0.0.0.0/0 → NAT Gateway

Private Subnet AZ-c
0.0.0.0/0 → NAT Gateway

#### 補足

実務では、AZ ごとに NAT Gateway を配置して冗長化する構成もある。
一方、本ハンズオンでは学習しやすさとのバランスを考慮し、Public Subnet 上の NAT Gateway を利用する前提としている。

### 6.ルート設計で確認したいポイント

本ハンズオンでは、ルート設計の理解を深めるために、以下の点を確認対象とする。

- Public Subnet のデフォルトルートが Internet Gateway を向いていること
- Private Subnet のデフォルトルートが NAT Gateway を向いていること
- VPC 内部通信は local ルートで処理されること
- Private EC2 から FSx for NetApp ONTAP への通信は NAT Gateway を経由しないこと
- Private EC2 からインターネットへの通信は NAT Gateway を経由すること

これらを整理することで、接続トラブル発生時に「どの経路を確認すべきか」を判断しやすくなる。

### 7.学習用としての位置づけ

未経験エンジニアが現場でつまずきやすいポイントの一つが、ルートテーブルがどの通信に影響するのか を正しく理解できていないことである。
そのため本ハンズオンでは、以下の 2 つを明確に区別できるようにしている。

- Private EC2 からインターネットへの通信
- Private EC2 から FSx for NetApp ONTAP への内部通信

この違いを理解できるようになることで、ネットワーク関連の切り分けや、疎通確認の考え方を実務に近い形で学べるようにしている。

## セキュリティ設計

本ハンズオンでは、管理用アクセスと業務用通信を分離し、必要最小限の通信のみを許可することを基本方針とする。  
未経験エンジニア向けの教材であっても、実務を意識して「とりあえず全部許可する」のではなく、**どのリソースに対して、どの通信を、どこから許可するのか** を明確にした設計とする。

本構成では、主に以下の通信を許可対象としている。

- Local PC から Bastion EC2 への SSH 通信
- Bastion EC2 から Private EC2 への SSH 通信
- Private EC2 から FSx for NetApp ONTAP への NFS 通信
- Private EC2 から外部へのアウトバウンド通信

---

### 1. セキュリティ設計の基本方針

本ハンズオンでは、以下の方針でセキュリティ設計を行う。

- Public に公開するのは Bastion EC2 のみとする
- Private EC2 は Public IP を持たせず、Bastion 経由でのみアクセス可能とする
- FSx for NetApp ONTAP は Private Subnet に配置し、Private EC2 からの NFS 通信のみを許可する
- 通信元は IP アドレス固定ではなく、可能な限り Security Group 単位で制御する
- 学習用であっても、役割ごとに許可通信を分離する

この方針により、接続経路が明確で、かつ実務でも通用しやすい構成を目指している。

---

### 2. Bastion EC2 のセキュリティ設計

Bastion EC2 は、ローカル PC からの管理用 SSH 接続を受け付ける踏み台サーバである。  
そのため、外部公開が必要な通信は SSH のみとする。

#### Inbound
- TCP 22
- Source: ローカル PC のグローバル IP

#### Outbound
- 全許可

#### 設計意図
Bastion は Public Subnet 上に配置するが、不要なポートは公開せず、SSH 接続のみに限定する。  
また、接続元は `0.0.0.0/0` ではなく、自分のグローバル IP を指定することで、学習用であっても最小限の公開範囲に抑える。

---

### 3. Private EC2 のセキュリティ設計

Private EC2 は、FSx for NetApp ONTAP のマウント確認や動作確認を行う検証用サーバである。  
外部から直接アクセスさせないため、SSH は Bastion 経由のみ許可する。

#### Inbound
- TCP 22
- Source: Bastion EC2 の Security Group

#### Outbound
- 全許可

#### 設計意図
Private EC2 は Public IP を持たず、Bastion からのみ SSH 接続できるようにする。  
これにより、管理用アクセスを制御しつつ、踏み台構成を理解しやすくしている。

また、Outbound を許可することで、以下の通信を可能にする。

- FSx for NetApp ONTAP への NFS 接続
- NAT Gateway 経由でのパッケージ取得
- DNS 解決のための通信

---

### 4. FSx for NetApp ONTAP のセキュリティ設計

FSx for NetApp ONTAP は共有ストレージであり、Private EC2 からの NFS 通信のみを許可する。  
本ハンズオンでは、SVM endpoint に対して NFS マウントを実施する。

#### Inbound
- TCP 2049
- Source: Private EC2 の Security Group

#### Outbound
- デフォルト設定を利用

#### 設計意図
FSx for NetApp ONTAP はインターネット公開するものではなく、VPC 内のクライアントからのみアクセスされる前提である。  
そのため、NFS で利用する TCP 2049 のみを Private EC2 から許可する。

ここで Source に Private EC2 の Security Group を指定することで、IP アドレスの変動に影響されにくく、実務でも扱いやすい構成としている。

---

### 5. Security Group 間の関係

本ハンズオンにおける Security Group の関係は以下の通りである。

Local PC
  ↓ TCP 22
Bastion SG
  ↓ TCP 22
Private EC2 SG
  ↓ TCP 2049
FSx SG

### 6.通信ごとの許可ルール

本ハンズオンで利用する主な通信と、対応するセキュリティルールは以下の通りである。

| 通信元         | 通信先                  | プロトコル / ポート | 許可方法                                  |
| ----------- | -------------------- | ----------- | ------------------------------------- |
| Local PC    | Bastion EC2          | TCP 22      | Bastion SG Inbound                    |
| Bastion EC2 | Private EC2          | TCP 22      | Private EC2 SG Inbound                |
| Private EC2 | FSx for NetApp ONTAP | TCP 2049    | FSx SG Inbound                        |
| Private EC2 | Internet             | 任意          | Private EC2 SG Outbound + NAT Gateway |

このように、各通信は Security Group で制御し、ルートテーブルとは役割を分けて考える。

### 7. Security Group とルートの役割分担

ネットワーク設計と混同しやすいポイントとして、Security Group とルートテーブルの違いがある。
本ハンズオンでは、以下のように整理する。

- ルートテーブル: 通信の行き先を決める
- Security Group: 通信を許可するかどうかを決める

例えば、Private EC2 から FSx for NetApp ONTAP への通信では、

- ルートは local
- 許可制御は FSx の Security Group

が担当する。

この違いを理解することで、疎通確認時に「通信経路の問題なのか」「許可設定の問題なのか」を切り分けやすくなる。

### 8. 学習上の確認ポイント

本ハンズオンでは、セキュリティ設計の理解を深めるために、以下の点を確認対象とする。

- Bastion には SSH のみを公開する
- Private EC2 は Bastion からの SSH のみ許可する
- FSx for NetApp ONTAP は Private EC2 からの NFS のみ許可する
- FSx への通信は IP ではなく Security Group ベースで制御する
- Security Group とルートテーブルは役割が異なる

これらを意識することで、未経験エンジニアでも接続設計とセキュリティ設計の違いを理解しやすくなる。

### 9. 学習用として簡略化している点

実務では、環境ごとに Security Group をさらに細かく分割したり、NACL や組織の標準ルールを組み合わせたりすることがある。
一方、本ハンズオンでは以下を優先してシンプルにしている。

- Bastion 用
- Private EC2 用
- FSx 用

という 3 つの役割に分けて Security Group を管理することで、未経験エンジニアでも通信経路と許可範囲を追いやすくしている。

このように、セキュリティ設計についても、学習しやすさを保ちながら実務での考え方を体験できる構成としている。

## 接続方式の設計

## 補足