# Kiro CLI EC2セットアップガイド

## 概要

本ガイドでは、AWS EC2インスタンス（AlmaLinux 9）上にKiro CLI開発環境を構築する手順を体系的に解説します。WindowsクライアントからVS Code Remote SSHで接続した状態で、Kiro CLIのインストールから認証・初期設定、GitLab連携までの一連のセットアップを行います。

## 対象読者

- AWS EC2上でKiro CLIを使った開発環境を構築したい開発者
- WindowsクライアントからVS Code Remote SSHでEC2に接続して開発を行う方
- AlmaLinux 9環境でのNode.js/CLIツールのセットアップに関心がある方

## 前提知識

- AWS EC2インスタンスの基本的な操作（起動・停止・接続）
- VS Code Remote SSHによるリモート接続が確立済みであること
- Linuxの基本的なコマンド操作（ターミナル操作、パッケージ管理）
- Gitの基本操作

---

## 目次

1. [前提条件](#1-前提条件)
   - [インスタンスタイプ・スペック要件](#インスタンスタイプスペック要件)
   - [対応OS](#対応os)
   - [ディスク容量要件](#ディスク容量要件)
   - [ネットワーク接続要件](#ネットワーク接続要件)
2. [Node.js環境構築](#2-nodejs環境構築)
   - [推奨バージョン](#推奨バージョン)
   - [dnf経由のインストール](#dnf経由のインストール)
   - [nvm経由のインストール（代替方法）](#nvm経由のインストール代替方法)
   - [バージョン確認](#バージョン確認)
3. [Kiro CLIインストール](#3-kiro-cliインストール)
   - [依存パッケージ](#依存パッケージ)
   - [インストール手順](#インストール手順)
   - [バージョン確認](#バージョン確認-1)
   - [パーミッションエラーへの対処](#パーミッションエラーへの対処)
4. [認証・初期設定](#4-認証初期設定)
   - [アカウント情報の準備](#アカウント情報の準備)
   - [デバイス認証フロー](#デバイス認証フロー)
   - [認証の確認](#認証の確認)
   - [認証失敗時のトラブルシューティング](#認証失敗時のトラブルシューティング)
5. [IAMロール設定](#5-iamロール設定)
   - [必要な権限](#必要な権限)
   - [IAMロールのアタッチ手順](#iamロールのアタッチ手順)
   - [最小権限ポリシー設定例](#最小権限ポリシー設定例)
   - [代替認証方法（アクセスキー）](#代替認証方法アクセスキー)
6. [GitLab連携設定](#6-gitlab連携設定)
   - [Git/SSH設定](#gitssh設定)
   - [GitLabへのSSH公開鍵登録](#gitlabへのssh公開鍵登録)
   - [プロジェクトのクローン](#プロジェクトのクローン)
7. [開発ワークフロー](#7-開発ワークフロー)
   - [主要コマンド一覧](#主要コマンド一覧)
   - [プロジェクトの初期化](#プロジェクトの初期化)
   - [specの作成・管理](#specの作成管理)
   - [VS Code Remote SSHでの開発フロー](#vs-code-remote-sshでの開発フロー)
8. [トラブルシューティング](#8-トラブルシューティング)
   - [Node.jsバージョン不整合](#nodejsバージョン不整合)
   - [Kiro CLI認証エラー](#kiro-cli認証エラー)
   - [EC2リソース不足](#ec2リソース不足)
   - [SELinux・ファイアウォール関連](#selinuxファイアウォール関連)
9. [GitLab+Kiro統合ワークフロー](#9-gitlabkiro統合ワークフロー)
   - [エンドツーエンドフロー概要](#エンドツーエンドフロー概要)
   - [ブランチ戦略とKiro CLIの連携パターン](#ブランチ戦略とkiro-cliの連携パターン)
   - [実践シナリオ: 機能開発のステップバイステップ](#実践シナリオ-機能開発のステップバイステップ)
   - [specファイルの運用・管理方針](#specファイルの運用管理方針)

---

## 1. 前提条件

本セクションでは、Kiro CLI開発環境を構築するためのEC2インスタンスの要件を記載します。以下の条件を満たしたインスタンスを用意してください。

> **重要:** 本ガイドは、WindowsクライアントからVS Code Remote SSHによるEC2インスタンスへの接続が既に確立されていることを前提としています。SSH接続の設定方法については、AWS公式ドキュメントおよびVS Code Remote SSH拡張機能のドキュメントを参照してください。

### インスタンスタイプ・スペック要件

Kiro CLIおよびNode.jsランタイムを快適に動作させるために、以下のスペックを推奨します。

| 項目 | 最小要件 | 推奨 |
|---|---|---|
| インスタンスタイプ | t3.small | **t3.medium 以上** |
| vCPU | 2 vCPU | 2 vCPU 以上 |
| メモリ | 4 GiB RAM | 4 GiB RAM 以上 |

- **t3.medium**（2 vCPU / 4 GiB RAM）を推奨します。Kiro CLIの実行に加え、Node.jsプロジェクトのビルドやテスト実行を考慮すると、4 GiB以上のメモリが必要です。
- t3.small（2 vCPU / 2 GiB RAM）でも動作可能ですが、大規模プロジェクトではメモリ不足が発生する可能性があります。

### 対応OS

| 項目 | 要件 |
|---|---|
| OS | **AlmaLinux 9** |
| アーキテクチャ | x86_64（amd64） |

- 本ガイドのすべての手順は **AlmaLinux 9** を対象として記載しています。
- 他のRHEL系ディストリビューション（Rocky Linux 9等）でも同様の手順で動作する可能性がありますが、検証対象外です。

### ディスク容量要件

| 項目 | 最小要件 | 推奨 |
|---|---|---|
| EBSボリュームサイズ | **20 GB** | 30 GB 以上 |

ディスク容量の内訳目安：

- OS・システムファイル: 約5 GB
- Node.js・npm: 約1 GB
- Kiro CLI・依存パッケージ: 約1 GB
- プロジェクトファイル・npmキャッシュ: 約5〜10 GB
- その他（ログ、一時ファイル等）: 約3〜5 GB

> **注意:** 複数のNode.jsプロジェクトを同時に扱う場合や、`node_modules` が大きいプロジェクトを扱う場合は30 GB以上を推奨します。

### ネットワーク接続要件

EC2インスタンスから**インターネットへのアウトバウンドアクセスが必須**です。以下の用途でインターネット接続が必要となります。

| 用途 | 接続先 | プロトコル |
|---|---|---|
| パッケージダウンロード（dnf） | AlmaLinuxリポジトリ | HTTPS（443） |
| Node.js/npmパッケージ取得 | registry.npmjs.org | HTTPS（443） |
| Kiro CLI認証 | Kiro認証サーバー | HTTPS（443） |
| GitLab接続（SSH） | gitlab.com（または自社GitLab） | SSH（22） |
| GitLab接続（HTTPS） | gitlab.com（または自社GitLab） | HTTPS（443） |

**セキュリティグループ設定の確認事項：**

- アウトバウンド: HTTPS（443）およびSSH（22）への送信を許可
- インバウンド: VS Code Remote SSH接続用にSSH（22）を許可（接続元IPを制限推奨）

> **注意:** プライベートサブネットに配置する場合は、NATゲートウェイまたはVPCエンドポイント経由でのインターネットアクセスを確保してください。

---

## 2. Node.js環境構築

Kiro CLIの実行にはNode.jsランタイムが必要です。本セクションでは、AlmaLinux 9上にNode.jsをインストールする2つの方法（dnf経由・nvm経由）を解説します。

### 推奨バージョン

| 項目 | 推奨 |
|---|---|
| Node.js | **v20 LTS**（Long Term Support） |
| npm | Node.js 20に同梱されるバージョン（v10以上） |

- Kiro CLIはNode.js 18以上で動作しますが、安定性とサポート期間の観点から **Node.js 20 LTS** を推奨します。
- Node.js 20 LTSは2026年4月までアクティブサポートが提供されます。

> **注意:** Node.js 16以下はサポート対象外です。必ず18以上のバージョンをインストールしてください。

### dnf経由のインストール

AlmaLinux 9のデフォルトリポジトリにはNode.js 20が含まれていないため、**NodeSourceリポジトリ**を追加してインストールします。

#### 1. 前提パッケージのインストール

```bash
sudo dnf install -y curl
```

#### 2. NodeSourceリポジトリの追加

NodeSource公式のセットアップスクリプトを使用して、Node.js 20用のリポジトリを追加します。

```bash
curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
```

> **補足:** 上記スクリプトは `/etc/yum.repos.d/nodesource-el9.repo` にリポジトリ設定ファイルを作成します。

#### 3. Node.jsのインストール

```bash
sudo dnf install -y nodejs
```

#### 4. インストール確認

```bash
node --version
# 出力例: v20.x.x

npm --version
# 出力例: 10.x.x
```

> **注意:** AlmaLinux 9のAppStreamリポジトリに含まれるNode.jsモジュール（`nodejs:18` 等）を使用する方法もありますが、最新のLTSバージョンを利用するためにNodeSourceリポジトリの追加を推奨します。

### nvm経由のインストール（代替方法）

nvm（Node Version Manager）を使用すると、複数のNode.jsバージョンを管理でき、sudoなしでインストールが可能です。プロジェクトごとに異なるNode.jsバージョンが必要な場合や、グローバルインストールのパーミッション問題を回避したい場合に推奨します。

#### 1. nvmのインストール

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

#### 2. シェル設定の読み込み

インストール後、nvmコマンドを有効にするためにシェル設定を再読み込みします。

```bash
source ~/.bashrc
```

#### 3. nvmの動作確認

```bash
nvm --version
# 出力例: 0.40.1
```

#### 4. Node.js 20 LTSのインストール

```bash
nvm install 20
```

#### 5. デフォルトバージョンの設定

```bash
nvm alias default 20
```

これにより、新しいシェルセッションでも自動的にNode.js 20が使用されます。

#### 6. インストール確認

```bash
node --version
# 出力例: v20.x.x

npm --version
# 出力例: 10.x.x
```

> **補足:** nvmで特定バージョンに切り替える場合は `nvm use <バージョン>` を使用します。例: `nvm use 20`

### バージョン確認

Node.jsのインストールが正常に完了したことを確認するために、以下のコマンドを実行してください。

```bash
# Node.jsのバージョン確認
node --version
```

**期待される出力:**

```
v20.x.x
```

```bash
# npmのバージョン確認
npm --version
```

**期待される出力:**

```
10.x.x
```

両方のコマンドでバージョン番号が正しく表示されれば、Node.js環境構築は完了です。

> **トラブルシューティング:**
> - `command not found: node` と表示される場合は、PATHが正しく設定されていない可能性があります。`source ~/.bashrc` を実行してから再試行してください。
> - nvm経由でインストールした場合、新しいターミナルセッションで `nvm use default` が自動的に実行されない場合は、`~/.bashrc` にnvmの初期化スクリプトが含まれていることを確認してください。

---

## 3. Kiro CLIインストール

Node.js環境が準備できたら、npm経由でKiro CLIをインストールします。本セクションでは、インストールに必要な依存パッケージ、インストール手順、およびよくあるパーミッションエラーへの対処方法を解説します。

### 依存パッケージ

Kiro CLIのインストールおよび動作には、以下のパッケージが必要です。

| パッケージ | バージョン要件 | 用途 |
|---|---|---|
| **Node.js** | 18以上（20 LTS推奨） | Kiro CLIランタイム |
| **npm** | Node.jsに同梱（v10以上推奨） | パッケージマネージャー |
| **git** | 2.x以上 | バージョン管理・プロジェクト操作 |

前セクションでNode.jsとnpmはインストール済みのため、gitがインストールされていることを確認します。

```bash
# gitがインストールされているか確認
git --version
```

**期待される出力:**

```
git version 2.x.x
```

gitがインストールされていない場合は、以下のコマンドでインストールしてください。

```bash
sudo dnf install -y git
```

### インストール手順

npmのグローバルインストールを使用してKiro CLIをインストールします。

```bash
npm install -g @anthropic-ai/kiro
```

> **補足:** グローバルインストール（`-g` オプション）により、ターミナルのどのディレクトリからでも `kiro` コマンドを実行できるようになります。

インストールが完了すると、以下のような出力が表示されます。

```
added XX packages in Xs
```

### バージョン確認

インストールが正常に完了したことを確認するために、バージョン確認コマンドを実行します。

```bash
kiro --version
```

**期待される出力例:**

```
x.x.x
```

バージョン番号が表示されれば、Kiro CLIのインストールは成功です。

> **トラブルシューティング:** `command not found: kiro` と表示される場合は、npmのグローバルインストール先がPATHに含まれていない可能性があります。以下のコマンドでnpmのグローバルインストール先を確認してください。
>
> ```bash
> npm prefix -g
> ```
>
> 表示されたパスの `bin` ディレクトリがPATHに含まれていることを確認してください。

### パーミッションエラーへの対処

npmでグローバルインストール（`npm install -g`）を実行した際に、以下のような `EACCES` パーミッションエラーが発生することがあります。

```
npm ERR! code EACCES
npm ERR! syscall mkdir
npm ERR! path /usr/lib/node_modules/@anthropic-ai/kiro
npm ERR! errno -13
npm ERR! Error: EACCES: permission denied, mkdir '/usr/lib/node_modules/@anthropic-ai/kiro'
```

**原因:** デフォルトのnpmグローバルインストール先（`/usr/lib/node_modules` 等）への書き込み権限がないために発生します。

> **⚠️ 注意:** `sudo npm install -g` で解決することも可能ですが、セキュリティ上推奨しません。以下のいずれかの方法で対処してください。

#### 方法1: npmのグローバルインストール先を変更する

ユーザーのホームディレクトリ配下にnpmのグローバルインストール先を変更することで、sudoなしでグローバルインストールが可能になります。

```bash
# 1. グローバルインストール用ディレクトリを作成
mkdir -p ~/.npm-global

# 2. npmのprefix設定を変更
npm config set prefix '~/.npm-global'

# 3. PATHに新しいディレクトリを追加（~/.bashrcに追記）
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc

# 4. シェル設定を再読み込み
source ~/.bashrc
```

設定が完了したら、再度Kiro CLIをインストールします。

```bash
npm install -g @anthropic-ai/kiro
```

**設定確認:**

```bash
# npmのグローバルインストール先が変更されたことを確認
npm prefix -g
# 出力例: /home/<ユーザー名>/.npm-global
```

#### 方法2: nvmを使用する（推奨）

前セクションでnvm経由でNode.jsをインストールした場合、この問題は発生しません。nvmはユーザーのホームディレクトリ（`~/.nvm/`）配下にNode.jsとnpmをインストールするため、グローバルインストール時にパーミッションの問題が起きません。

nvmをまだインストールしていない場合は、[nvm経由のインストール（代替方法）](#nvm経由のインストール代替方法)を参照してnvmをセットアップし、その後Kiro CLIをインストールしてください。

```bash
# nvmでNode.jsをインストール済みの場合、そのままインストール可能
npm install -g @anthropic-ai/kiro

# パーミッションエラーなくインストールされることを確認
kiro --version
```

> **補足:** nvmを使用する方法は、複数のNode.jsバージョンを管理する場合にも便利です。プロジェクトごとに異なるNode.jsバージョンが必要な場合は、nvm経由のセットアップを推奨します。

---

## 4. 認証・初期設定

Kiro CLIを使用するには、認証（ログイン）が必要です。EC2インスタンスのようなヘッドレス環境（ブラウザが使えない環境）では、**デバイス認証フロー**を使用してログインを行います。本セクションでは、アカウントの準備から認証完了までの手順を解説します。

### アカウント情報の準備

Kiro CLIの認証には、以下のいずれかのアカウントが必要です。

| 認証方法 | 説明 | 推奨用途 |
|---|---|---|
| **AWS Builder ID** | 個人向けの無料アカウント | 個人開発、学習用途 |
| **IAM Identity Center** | 組織管理のアカウント | 企業・チーム開発 |

#### AWS Builder IDを使用する場合

AWS Builder IDをまだ持っていない場合は、以下の手順で作成してください。

1. [AWS Builder ID登録ページ](https://profile.aws.amazon.com/) にアクセス
2. 「Create AWS Builder ID」を選択
3. メールアドレスを入力し、確認コードを受信
4. 名前とパスワードを設定して登録完了

> **補足:** AWS Builder IDは無料で作成でき、AWSアカウント（請求が発生するアカウント）とは別のものです。Kiro CLIの基本機能を利用するために追加料金は発生しません。

#### IAM Identity Centerを使用する場合

組織のIAM Identity Center（旧AWS SSO）を使用する場合は、組織の管理者から以下の情報を取得してください。

- IAM Identity CenterのスタートURL（例: `https://your-org.awsapps.com/start`）
- 使用するリージョン

### デバイス認証フロー

EC2インスタンス上のターミナルはブラウザが利用できないため、**デバイス認証フロー**（Device Authorization Flow）を使用します。この方式では、ターミナルに表示されるURLとコードを、手元のPC（Windows側）のブラウザで開いて認証を完了します。

#### 認証手順

##### 1. `kiro login` コマンドを実行

EC2インスタンスのターミナルで以下のコマンドを実行します。

```bash
kiro login
```

##### 2. 認証URLとユーザーコードを確認

コマンドを実行すると、ターミナルに以下のような出力が表示されます。

```
To sign in, open the following URL in your browser:

  https://device.sso.us-east-1.amazonaws.com/

And enter the code:

  ABCD-EFGH
```

以下の2つの情報をメモしてください。

- **認証URL**: ブラウザで開くURL
- **ユーザーコード**: 認証画面で入力するコード（例: `ABCD-EFGH`）

> **注意:** ユーザーコードには有効期限があります（通常数分間）。期限切れになった場合は、再度 `kiro login` を実行して新しいコードを取得してください。

##### 3. ブラウザで認証を完了

手元のPC（Windows側）で以下の操作を行います。

1. ブラウザを開き、ターミナルに表示された **認証URL** にアクセス
2. 画面にユーザーコードの入力欄が表示されるので、ターミナルに表示された **ユーザーコード** を入力
3. 「Confirm and continue」（確認して続行）をクリック
4. AWS Builder IDまたはIAM Identity Centerの認証情報でサインイン
5. アクセス許可の確認画面が表示されたら「Allow」（許可）をクリック

##### 4. ターミナルで認証完了を確認

ブラウザでの認証が完了すると、EC2ターミナル側に認証成功のメッセージが自動的に表示されます。

```
Successfully logged in!
```

これでKiro CLIの認証は完了です。

> **補足:** 認証情報はローカルに保存されるため、通常は一度ログインすれば継続的に利用できます。トークンの有効期限が切れた場合は、再度 `kiro login` を実行してください。

### 認証の確認

認証が正しく完了しているかを確認するには、Kiro CLIのコマンドを実行してみてください。

```bash
kiro --version
```

認証済みの状態でKiro CLIのコマンドが正常に動作すれば、認証は成功しています。

また、認証エラーが発生する場合は、再度 `kiro login` を実行してログインし直してください。

```bash
# 認証状態をリフレッシュする場合
kiro login
```

### 認証失敗時のトラブルシューティング

認証時に問題が発生した場合は、以下の一般的なケースと対処法を参照してください。

#### 問題1: デバイスコードの有効期限切れ

**症状:** ブラウザでコードを入力した際に「コードが無効です」「期限切れです」などのエラーが表示される。

**原因:** `kiro login` 実行後、ブラウザでの認証操作に時間がかかり、ユーザーコードの有効期限が切れた。

**解決方法:**

1. ターミナルに戻り、`Ctrl + C` で現在のプロセスをキャンセル
2. 再度 `kiro login` を実行して新しいコードを取得
3. 今度は速やかにブラウザで認証を完了する

```bash
# 再度ログインを実行
kiro login
```

> **Tips:** 認証URLとコードが表示されたら、すぐにブラウザで操作を開始してください。通常5〜10分程度の有効期限があります。

#### 問題2: ネットワーク接続エラー

**症状:** `kiro login` 実行時に接続エラーやタイムアウトが発生する。

```
Error: connect ETIMEDOUT
```
または
```
Error: getaddrinfo ENOTFOUND device.sso.us-east-1.amazonaws.com
```

**原因:** EC2インスタンスからKiro認証サーバーへのネットワーク接続ができていない。

**解決方法:**

1. インターネット接続を確認

   ```bash
   # 外部への接続確認
   curl -I https://aws.amazon.com
   ```

2. DNS解決を確認

   ```bash
   # DNS解決の確認
   nslookup device.sso.us-east-1.amazonaws.com
   ```

3. セキュリティグループのアウトバウンドルールを確認
   - HTTPS（ポート443）へのアウトバウンド通信が許可されていることを確認

4. プライベートサブネットの場合、NATゲートウェイが正しく設定されていることを確認

#### 問題3: 認証トークンの期限切れ

**症状:** 以前は正常に動作していたKiro CLIが、突然認証エラーを返すようになった。

```
Error: Token has expired
```
または
```
Error: Authentication required. Please run 'kiro login'.
```

**原因:** 保存されている認証トークンの有効期限が切れた。

**解決方法:**

再度ログインを実行して、トークンをリフレッシュします。

```bash
kiro login
```

上記のデバイス認証フローに従って再認証を完了してください。

#### 問題4: ブラウザが開けない（ヘッドレス環境）

**症状:** `kiro login` 実行時に「ブラウザを開けません」というメッセージが表示される。

**原因:** EC2インスタンスはGUI環境がないため、自動的にブラウザを起動できない。これは正常な動作です。

**解決方法:**

ターミナルに表示された認証URLを手動でコピーし、手元のPC（Windows側）のブラウザに貼り付けて開いてください。デバイス認証フローはこの使い方を想定しています。

> **まとめ:** 認証に関する問題の多くは、`kiro login` を再実行することで解決できます。ネットワーク接続に問題がある場合は、セキュリティグループとインターネット接続を先に確認してください。

---

## 5. IAMロール設定

Kiro CLIがAWSサービス（Amazon Q / CodeWhisperer関連）と連携するために、EC2インスタンスに適切なIAMロールを設定します。IAMロールを使用することで、アクセスキーを直接管理する必要がなくなり、セキュリティが向上します。

### 必要な権限

Kiro CLIの動作には、以下のAWSサービスへのアクセス権限が必要です。

| サービス | 用途 | 必要なアクション |
|---|---|---|
| Amazon Q Developer | コード補完・提案機能 | `q:SendMessage`, `q:GetConversation` |
| CodeWhisperer | AIコード生成 | `codewhisperer:GenerateCompletions`, `codewhisperer:GenerateRecommendations` |
| AWS SSO (Identity Center) | 認証連携 | `sso:GetRoleCredentials` |
| STS | 認証情報の確認 | `sts:GetCallerIdentity` |

> **注意:** Kiro CLIの認証は主にBuilder ID（個人認証）またはIAM Identity Center経由で行われます。ここで設定するIAMロールは、Kiro CLIがAWSリソースにアクセスする際の追加的な権限付与のために使用されます。

### IAMロールのアタッチ手順

#### 方法1: AWSマネジメントコンソールから設定

1. **IAMロールの作成**
   - AWSマネジメントコンソールで「IAM」→「ロール」→「ロールを作成」を選択
   - 信頼されたエンティティタイプ: 「AWSサービス」を選択
   - ユースケース: 「EC2」を選択
   - 「次へ」をクリック

2. **ポリシーのアタッチ**
   - 「ポリシーを作成」で後述のカスタムポリシーを作成するか、既存ポリシーを選択
   - ポリシー名を入力（例: `KiroCLI-EC2-Policy`）
   - ロール名を入力（例: `KiroCLI-EC2-Role`）
   - 「ロールを作成」をクリック

3. **EC2インスタンスへのロールアタッチ**
   - 「EC2」→「インスタンス」で対象インスタンスを選択
   - 「アクション」→「セキュリティ」→「IAMロールを変更」を選択
   - 作成したロール（`KiroCLI-EC2-Role`）を選択
   - 「IAMロールの更新」をクリック

#### 方法2: AWS CLIから設定

```bash
# 1. IAMロールの信頼ポリシーを作成
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 2. IAMロールを作成
aws iam create-role \
  --role-name KiroCLI-EC2-Role \
  --assume-role-policy-document file://trust-policy.json \
  --description "IAM role for Kiro CLI on EC2"

# 3. カスタムポリシーを作成してアタッチ（ポリシーJSONは次セクション参照）
aws iam put-role-policy \
  --role-name KiroCLI-EC2-Role \
  --policy-name KiroCLI-Permissions \
  --policy-document file://kiro-cli-policy.json

# 4. インスタンスプロファイルを作成
aws iam create-instance-profile \
  --instance-profile-name KiroCLI-EC2-Profile

# 5. ロールをインスタンスプロファイルに追加
aws iam add-role-to-instance-profile \
  --instance-profile-name KiroCLI-EC2-Profile \
  --role-name KiroCLI-EC2-Role

# 6. EC2インスタンスにインスタンスプロファイルをアタッチ
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxxxxxxxxxxxxxxx \
  --iam-instance-profile Name=KiroCLI-EC2-Profile
```

> **注意:** `i-xxxxxxxxxxxxxxxxx` は実際のEC2インスタンスIDに置き換えてください。

### 最小権限ポリシー設定例

以下は最小権限の原則に基づいたIAMポリシーの設定例です。`kiro-cli-policy.json` として保存してください。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KiroCLICodeWhispererAccess",
      "Effect": "Allow",
      "Action": [
        "codewhisperer:GenerateCompletions",
        "codewhisperer:GenerateRecommendations",
        "codewhisperer:ListRecommendations"
      ],
      "Resource": "*"
    },
    {
      "Sid": "KiroCLIAmazonQAccess",
      "Effect": "Allow",
      "Action": [
        "q:SendMessage",
        "q:GetConversation",
        "q:StartConversation",
        "q:ListConversations"
      ],
      "Resource": "*"
    },
    {
      "Sid": "STSIdentityVerification",
      "Effect": "Allow",
      "Action": [
        "sts:GetCallerIdentity"
      ],
      "Resource": "*"
    }
  ]
}
```

> **セキュリティのポイント:**
> - `Resource` は可能な限り特定のARNに制限してください。上記例では `"*"` を使用していますが、組織のポリシーに応じてリソース範囲を絞り込むことを推奨します。
> - 不要になった権限は速やかに削除してください。
> - 定期的にIAM Access Analyzerを使用して、未使用の権限を特定・削除してください。

### 確認コマンド

IAMロールが正しく設定されていることを確認します。

```bash
# IAMロールがアタッチされているか確認
aws sts get-caller-identity
```

**期待される出力例:**

```json
{
    "UserId": "AROAXXXXXXXXXXXXXXXXX:i-xxxxxxxxxxxxxxxxx",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/KiroCLI-EC2-Role/i-xxxxxxxxxxxxxxxxx"
}
```

- `Arn` にロール名（`KiroCLI-EC2-Role`）が含まれていれば、IAMロールが正しくアタッチされています。
- エラー `Unable to locate credentials` が表示される場合は、IAMロールが正しくアタッチされていません。上記の手順を再確認してください。

```bash
# アタッチされたポリシーの確認
aws iam list-role-policies --role-name KiroCLI-EC2-Role
aws iam list-attached-role-policies --role-name KiroCLI-EC2-Role
```

### 代替認証方法（アクセスキー）

IAMロールをEC2インスタンスにアタッチできない環境（権限不足、組織ポリシー等）では、IAMユーザーのアクセスキーを使用して認証を行うことができます。

> **⚠️ セキュリティ上の注意:** アクセスキーはIAMロールと比較してセキュリティリスクが高くなります。可能な限りIAMロールの使用を推奨します。アクセスキーを使用する場合は、以下の点に注意してください。
> - アクセスキーを定期的にローテーションする（90日以内推奨）
> - 不要になったアクセスキーは速やかに無効化・削除する
> - アクセスキーをソースコードやGitリポジトリにコミットしない

#### アクセスキーの設定手順

1. **IAMユーザーのアクセスキーを作成**（AWSコンソールまたはCLI）

   ```bash
   # IAMコンソールで作成する場合:
   # IAM → ユーザー → 対象ユーザー → セキュリティ認証情報 → アクセスキーを作成
   ```

2. **AWS CLIの認証情報を設定**

   ```bash
   aws configure
   ```

   以下の情報を入力します：

   ```
   AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
   AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   Default region name [None]: ap-northeast-1
   Default output format [None]: json
   ```

   > **注意:** 上記のアクセスキーはサンプルです。実際のアクセスキーを使用してください。

3. **設定の確認**

   ```bash
   # 認証情報が正しく設定されているか確認
   aws sts get-caller-identity
   ```

   **期待される出力例:**

   ```json
   {
       "UserId": "AIDAXXXXXXXXXXXXXXXXX",
       "Account": "123456789012",
       "Arn": "arn:aws:iam::123456789012:user/kiro-cli-user"
   }
   ```

4. **認証情報ファイルの確認**

   ```bash
   # 設定が保存されていることを確認
   cat ~/.aws/credentials
   cat ~/.aws/config
   ```

> **Tips:** 複数のAWSプロファイルを使い分ける場合は、`aws configure --profile kiro` でプロファイルを作成し、コマンド実行時に `--profile kiro` を指定するか、環境変数 `AWS_PROFILE=kiro` を設定してください。

---

## 6. GitLab連携設定

EC2インスタンスからGitLabリポジトリにSSH経由でアクセスするための設定を行います。Git クライアントの設定、SSH鍵の生成、GitLabへの公開鍵登録、プロジェクトのクローンまでの一連の手順を解説します。

### Git/SSH設定

#### 1. Gitクライアントのインストール確認

AlmaLinux 9ではGitが標準で利用可能ですが、インストールされていない場合は以下のコマンドでインストールします。

```bash
# Gitがインストールされているか確認
git --version
```

**期待される出力:**

```
git version 2.x.x
```

`command not found` と表示される場合は、以下のコマンドでインストールしてください。

```bash
sudo dnf install -y git
```

#### 2. Gitユーザー情報の設定

コミット時に使用するユーザー名とメールアドレスを設定します。GitLabアカウントと同じ情報を設定してください。

```bash
# ユーザー名を設定
git config --global user.name "あなたの名前"

# メールアドレスを設定（GitLabアカウントのメールアドレス）
git config --global user.email "your-email@example.com"
```

設定内容を確認します。

```bash
git config --global --list
```

**期待される出力例:**

```
user.name=あなたの名前
user.email=your-email@example.com
```

> **補足:** `--global` オプションにより、このEC2インスタンス上のすべてのGitリポジトリで共通の設定となります。プロジェクトごとに異なる設定を使用する場合は、リポジトリ内で `--global` を外して実行してください。

#### 3. SSH鍵の生成

GitLabとのSSH接続に使用する鍵ペア（秘密鍵・公開鍵）を生成します。Ed25519アルゴリズムを推奨します。

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

実行すると以下のプロンプトが表示されます。

```
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/<ユーザー名>/.ssh/id_ed25519):
```

- **保存先:** デフォルト（`~/.ssh/id_ed25519`）のままEnterキーを押します
- **パスフレーズ:** セキュリティ向上のため、パスフレーズの設定を推奨します（空でも可）

```
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

鍵が正常に生成されると、以下のような出力が表示されます。

```
Your identification has been saved in /home/<ユーザー名>/.ssh/id_ed25519
Your public key has been saved in /home/<ユーザー名>/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx your-email@example.com
```

> **注意:** Ed25519がサポートされていない環境の場合は、RSA 4096ビットを使用してください。
> ```bash
> ssh-keygen -t rsa -b 4096 -C "your-email@example.com"
> ```

#### 4. ssh-agentの設定

パスフレーズ付きのSSH鍵を使用する場合、ssh-agentに鍵を登録しておくと毎回パスフレーズを入力する手間が省けます。

```bash
# ssh-agentをバックグラウンドで起動
eval "$(ssh-agent -s)"
```

**期待される出力:**

```
Agent pid 12345
```

```bash
# SSH秘密鍵をagentに追加
ssh-add ~/.ssh/id_ed25519
```

パスフレーズを設定した場合は入力を求められます。

> **補足:** シェルセッション起動時に自動でssh-agentを起動するには、`~/.bashrc` に以下を追記してください。
> ```bash
> # ssh-agent自動起動
> if [ -z "$SSH_AUTH_SOCK" ]; then
>   eval "$(ssh-agent -s)" > /dev/null 2>&1
>   ssh-add ~/.ssh/id_ed25519 2> /dev/null
> fi
> ```

### GitLabへのSSH公開鍵登録

生成したSSH公開鍵をGitLabに登録することで、パスワードなしでリポジトリにアクセスできるようになります。

#### 1. 公開鍵の内容をコピー

```bash
cat ~/.ssh/id_ed25519.pub
```

**出力例:**

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... your-email@example.com
```

この出力内容を全文コピーしてください。

#### 2. GitLabでSSH公開鍵を登録

1. ブラウザでGitLabにログインします
2. 右上のアバターアイコンをクリック →「**Preferences**」（設定）を選択
3. 左サイドバーから「**SSH Keys**」を選択
4. 「**Add new key**」ボタンをクリック
5. 以下の情報を入力します：

| 項目 | 入力内容 |
|---|---|
| **Key** | コピーした公開鍵の内容を貼り付け |
| **Title** | 鍵を識別する名前（例: `EC2-AlmaLinux9-dev`） |
| **Usage type** | 「Authentication & Signing」（デフォルト） |
| **Expiration date** | 必要に応じて有効期限を設定（任意） |

6. 「**Add key**」ボタンをクリックして登録を完了します

> **セキュリティ推奨:** 鍵に有効期限を設定し、定期的にローテーションすることを推奨します。組織のポリシーに従い、適切な有効期限（例: 1年）を設定してください。

#### 3. 登録確認

SSH Keys一覧に新しく追加した鍵が表示されていることを確認してください。

### プロジェクトのクローン

#### 1. SSH接続の確認

GitLabへのSSH接続が正常に動作するか確認します。

```bash
ssh -T git@gitlab.com
```

初回接続時にホスト鍵の確認を求められる場合があります。

```
The authenticity of host 'gitlab.com (xxx.xxx.xxx.xxx)' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

`yes` と入力してEnterキーを押してください。

**成功時の出力:**

```
Welcome to GitLab, @your-username!
```

このメッセージが表示されれば、SSH接続は正常に設定されています。

> **自社GitLabを使用する場合:** `gitlab.com` を自社GitLabサーバーのホスト名に置き換えてください。
> ```bash
> ssh -T git@gitlab.your-company.com
> ```

> **トラブルシューティング:**
> - `Permission denied (publickey)` と表示される場合:
>   - SSH公開鍵がGitLabに正しく登録されているか確認してください
>   - `ssh-add -l` で鍵がagentに登録されているか確認してください
>   - `ssh -vT git@gitlab.com` で詳細なデバッグ情報を確認できます
> - `Host key verification failed` と表示される場合:
>   - 以下のコマンドでホスト鍵を手動で追加してください
>   ```bash
>   ssh-keyscan gitlab.com >> ~/.ssh/known_hosts
>   ```

#### 2. プロジェクトのクローン

SSH接続が確認できたら、GitLabプロジェクトをクローンします。

```bash
# 作業ディレクトリに移動（例: ホームディレクトリ配下にprojectsディレクトリを作成）
mkdir -p ~/projects
cd ~/projects

# GitLabプロジェクトをクローン（SSH URL）
git clone git@gitlab.com:group/project.git
```

> **SSH URLの確認方法:** GitLabのプロジェクトページで「**Code**」ボタンをクリックし、「**Clone with SSH**」のURLをコピーしてください。

**クローン成功時の出力例:**

```
Cloning into 'project'...
remote: Enumerating objects: 150, done.
remote: Counting objects: 100% (150/150), done.
remote: Compressing objects: 100% (100/100), done.
remote: Total 150 (delta 50), reused 130 (delta 40), pack-reused 0
Receiving objects: 100% (150/150), 1.50 MiB | 5.00 MiB/s, done.
Resolving deltas: 100% (50/50), done.
```

```bash
# クローンしたディレクトリに移動
cd project

# リポジトリの状態を確認
git status
```

**期待される出力:**

```
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

> **自社GitLabの場合のSSH URL形式:**
> ```
> git@gitlab.your-company.com:group-name/project-name.git
> ```

これでGitLab連携設定は完了です。次のセクション「開発ワークフロー」で、Kiro CLIとGitLabを組み合わせた開発フローを解説します。

---

## 7. 開発ワークフロー

本セクションでは、Kiro CLIの基本的な操作方法と、VS Code Remote SSH環境での開発ワークフローを解説します。Kiro CLIを使うことで、仕様（spec）駆動の開発プロセスを効率的に進めることができます。

### 主要コマンド一覧

Kiro CLIで利用可能な主要コマンドの一覧です。

| コマンド | 説明 | 使用例 |
|---|---|---|
| `kiro init` | プロジェクトにKiro環境を初期化する | `kiro init` |
| `kiro spec` | specの作成・管理を行う | `kiro spec` |
| `kiro login` | Kiro CLIの認証（ログイン）を行う | `kiro login` |
| `kiro --version` | インストールされているKiro CLIのバージョンを表示する | `kiro --version` |
| `kiro --help` | ヘルプメッセージとコマンド一覧を表示する | `kiro --help` |

> **補足:** 各コマンドのサブオプションを確認するには、`kiro <コマンド> --help` を実行してください。

### プロジェクトの初期化

Kiro CLIを使って開発を始めるには、まずプロジェクトディレクトリでKiro環境を初期化します。

#### 1. プロジェクトディレクトリへ移動

GitLabからクローンしたプロジェクト、または新規作成したプロジェクトのルートディレクトリに移動します。

```bash
cd ~/projects/your-project
```

#### 2. `kiro init` を実行

```bash
kiro init
```

#### 3. 作成されるディレクトリ構造

`kiro init` を実行すると、プロジェクトルートに `.kiro/` ディレクトリが作成され、以下の構造が生成されます。

```
your-project/
├── .kiro/
│   ├── specs/          # spec（仕様）ファイルの格納先
│   ├── steering/       # ステアリングファイル（プロジェクト設定）
│   └── settings.json   # Kiroプロジェクト設定
├── src/
├── ...
└── README.md
```

| ディレクトリ/ファイル | 説明 |
|---|---|
| `.kiro/specs/` | specファイルを格納するディレクトリ。各specはサブディレクトリとして管理される |
| `.kiro/steering/` | Kiroの動作を制御するステアリングファイルを格納 |
| `.kiro/settings.json` | プロジェクト固有のKiro設定 |

> **注意:** `.kiro/` ディレクトリはGitリポジトリにコミットして、チームメンバーと共有することを推奨します。specやステアリング設定をチーム全体で統一できます。

#### 4. 初期化の確認

```bash
# .kiroディレクトリが作成されたことを確認
ls -la .kiro/
```

**期待される出力例:**

```
total 12
drwxr-xr-x 4 user user 4096 ...  .
drwxr-xr-x 5 user user 4096 ...  ..
drwxr-xr-x 2 user user 4096 ...  specs
drwxr-xr-x 2 user user 4096 ...  steering
-rw-r--r-- 1 user user  ... ...  settings.json
```

### specの作成・管理

Kiro CLIの中心的な機能であるspec（仕様）は、開発タスクを構造化して管理するための仕組みです。specを使うことで、要件定義→設計→タスク分解の流れを一貫して管理できます。

#### specとは

specはKiroの仕様駆動開発の基本単位です。1つのspecは以下の3つのフェーズで構成されます。

| フェーズ | ファイル | 説明 |
|---|---|---|
| **Requirements**（要件定義） | `requirements.md` | ユーザーストーリーと受入基準を定義 |
| **Design**（設計） | `design.md` | アーキテクチャ、コンポーネント設計、データモデルを定義 |
| **Tasks**（タスク） | `tasks.md` | 実装タスクの分解と実行計画を定義 |

#### specの作成手順

##### 1. `kiro spec` コマンドを実行

プロジェクトルートで以下のコマンドを実行します。

```bash
kiro spec
```

##### 2. specの種類を選択

対話形式でspecの種類や名前を指定します。プロンプトに従って必要な情報を入力してください。

##### 3. 作成されるファイル構造

specを作成すると、`.kiro/specs/` 配下に以下のようなディレクトリ構造が生成されます。

```
.kiro/specs/
└── your-spec-name/
    ├── requirements.md    # 要件定義
    ├── design.md          # 設計ドキュメント
    └── tasks.md           # 実装タスク一覧
```

#### specのワークフロー

specを活用した開発は、以下の流れで進めます。

```
1. Requirements（要件定義）
   └── ユーザーストーリー・受入基準を記載

2. Design（設計）
   └── アーキテクチャ・コンポーネント・データモデルを設計

3. Tasks（タスク分解）
   └── 実装タスクを具体的なステップに分解

4. Implementation（実装）
   └── タスクに沿ってコードを実装
```

> **ポイント:** specのワークフローでは、Requirements → Design → Tasks の順にドキュメントが段階的に作成されます。各フェーズで内容をレビューし、次のフェーズに進むことで、手戻りの少ない開発が可能です。

#### specの管理

作成済みのspecを確認・管理するには、`.kiro/specs/` ディレクトリの内容を確認します。

```bash
# 作成済みのspec一覧を確認
ls .kiro/specs/
```

各specのタスク進捗は、`tasks.md` ファイル内のチェックボックスで管理されます。

```markdown
- [x] 完了したタスク
- [ ] 未完了のタスク
```

### VS Code Remote SSHでの開発フロー

VS Code Remote SSHでEC2に接続した状態での、specを活用した一連の開発フローを解説します。

#### 全体の流れ

```
1. specの作成（要件・設計・タスク定義）
     ↓
2. ブランチの作成
     ↓
3. コードの実装（タスクに沿って）
     ↓
4. テスト・動作確認
     ↓
5. Gitコミット・プッシュ
     ↓
6. マージリクエスト（MR）の作成
```

#### ステップ1: specの作成

VS Codeのターミナルを開き（`` Ctrl+` ``）、プロジェクトルートでspecを作成します。

```bash
# プロジェクトルートに移動
cd ~/projects/your-project

# specを作成
kiro spec
```

作成されたspecファイル（`.kiro/specs/<spec名>/requirements.md` 等）をVS Codeのエディタで開き、要件・設計・タスクを記載します。

#### ステップ2: フィーチャーブランチの作成

specで定義したタスクの実装を始める前に、フィーチャーブランチを作成します。

```bash
# mainブランチを最新に更新
git checkout main
git pull origin main

# フィーチャーブランチを作成
git checkout -b feature/your-feature-name
```

> **ブランチ命名規則の例:**
> - `feature/機能名` — 新機能の追加
> - `fix/バグ内容` — バグ修正
> - `docs/ドキュメント名` — ドキュメント追加・修正

#### ステップ3: コードの実装

specのtasks.mdに定義されたタスクに沿って、コードを実装します。VS Codeのエディタを活用し、Remote SSH環境で直接ファイルを編集できます。

```bash
# ファイルの作成・編集はVS Codeのエディタで行う
# ターミナルでの操作が必要な場合:
mkdir -p src/components
touch src/components/new-feature.ts
```

タスクが完了したら、tasks.mdのチェックボックスを更新します。

```markdown
- [x] 完了したタスク
- [ ] 次のタスク
```

#### ステップ4: テスト・動作確認

実装が完了したら、テストを実行して動作を確認します。

```bash
# プロジェクトに応じたテストコマンド（例）
npm test
# または
npm run build
```

#### ステップ5: Gitコミット・プッシュ

変更内容をコミットし、リモートリポジトリにプッシュします。

```bash
# 変更状態を確認
git status

# 変更ファイルをステージング
git add .

# コミット（specのタスクに対応したメッセージを推奨）
git commit -m "feat: タスク名に対応する実装内容の説明"

# リモートリポジトリにプッシュ
git push -u origin feature/your-feature-name
```

> **コミットメッセージの推奨フォーマット:**
> ```
> <種別>: <変更の要約>
>
> - 変更内容の詳細1
> - 変更内容の詳細2
>
> Refs: spec/<spec名>
> ```
>
> 種別: `feat`（機能追加）、`fix`（修正）、`docs`（ドキュメント）、`refactor`（リファクタリング）

#### ステップ6: マージリクエスト（MR）の作成

GitLabでマージリクエストを作成し、コードレビューを依頼します。

##### 方法1: GitLab WebUIから作成

1. ブラウザでGitLabのプロジェクトページを開く
2. プッシュ直後に表示される「Create merge request」のバナーをクリック
   - またはサイドバーの「Merge requests」→「New merge request」から作成
3. 以下の情報を入力：

| 項目 | 入力内容 |
|---|---|
| **Source branch** | `feature/your-feature-name` |
| **Target branch** | `main`（または開発ブランチ） |
| **Title** | MRのタイトル（例: `feat: 新機能の実装`） |
| **Description** | 変更内容の説明、関連するspec情報 |
| **Assignee** | レビュー担当者 |

4. 「Create merge request」をクリック

##### 方法2: ターミナルから作成（GitLab CLI使用）

GitLab CLI（`glab`）がインストールされている場合は、ターミナルからMRを作成することもできます。

```bash
# GitLab CLIでMRを作成
glab mr create \
  --title "feat: タスク名に対応する実装" \
  --description "## 変更内容\n- 実装内容の説明\n\n## 関連spec\n- .kiro/specs/your-spec-name/" \
  --target-branch main
```

> **Tips:**
> - MRの説明にspecへの参照を含めることで、レビュアーが要件・設計を確認しやすくなります
> - レビュー完了後、GitLab上で「Merge」ボタンをクリックしてマージします
> - マージ後はローカルのブランチを整理してください
>   ```bash
>   git checkout main
>   git pull origin main
>   git branch -d feature/your-feature-name
>   ```

---

## 8. トラブルシューティング

本セクションでは、Kiro CLI開発環境の構築・運用時に発生しやすい問題と、その解決方法を体系的にまとめています。各問題は「症状→原因→解決方法→確認コマンド」の統一フォーマットで記載しています。

### Node.jsバージョン不整合

#### 問題: `command not found: node` と表示される

**原因:** Node.jsがインストールされていないか、インストール先のパスがシェルのPATH環境変数に含まれていない。nvm経由でインストールした場合、新しいシェルセッションでnvmの初期化が行われていない可能性がある。

**解決方法:**

1. nvmを使用している場合、シェル設定を再読み込みする

   ```bash
   source ~/.bashrc
   ```

2. nvmの初期化スクリプトが `~/.bashrc` に含まれているか確認する

   ```bash
   grep -n "nvm" ~/.bashrc
   ```

3. 含まれていない場合、以下を `~/.bashrc` に追記する

   ```bash
   export NVM_DIR="$HOME/.nvm"
   [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
   [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"
   ```

4. dnf経由でインストールした場合は、パスを確認する

   ```bash
   which node || find / -name "node" -type f 2>/dev/null
   ```

**確認コマンド:**

```bash
source ~/.bashrc
node --version
echo $PATH | tr ':' '\n' | grep -E "(nvm|node)"
```

#### 問題: Node.jsのバージョンがKiro CLIの要件を満たさない

**原因:** AlmaLinux 9のAppStreamリポジトリからインストールした場合、Node.js 16等の古いバージョンがインストールされることがある。Kiro CLIはNode.js 18以上を要求する。

**解決方法:**

1. 現在のNode.jsバージョンを確認する

   ```bash
   node --version
   ```

2. 古いバージョンの場合、NodeSourceリポジトリ経由でNode.js 20をインストールする

   ```bash
   # 既存のNode.jsを削除
   sudo dnf remove -y nodejs

   # NodeSourceリポジトリを追加してNode.js 20をインストール
   curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
   sudo dnf install -y nodejs
   ```

3. nvmを使用している場合は、バージョンを切り替える

   ```bash
   nvm install 20
   nvm alias default 20
   nvm use 20
   ```

**確認コマンド:**

```bash
node --version
# v20.x.x と表示されることを確認
npm --version
# 10.x.x と表示されることを確認
```

#### 問題: 複数のNode.jsバージョンが競合している

**原因:** dnf経由とnvm経由の両方でNode.jsをインストールした結果、PATHの優先順位によって意図しないバージョンが使用されている。

**解決方法:**

1. どのNode.jsが使用されているか確認する

   ```bash
   which node
   type -a node
   ```

2. dnf経由のNode.jsが不要な場合は削除する

   ```bash
   sudo dnf remove -y nodejs
   ```

3. nvmを優先する場合は、シェル設定でnvmのパスが先に読み込まれることを確認する

   ```bash
   # ~/.bashrcの末尾にnvm初期化が記載されていることを確認
   tail -10 ~/.bashrc
   ```

4. シェルを再起動して設定を反映する

   ```bash
   exec bash
   ```

**確認コマンド:**

```bash
which node
node --version
npm --version
```

---

### Kiro CLI認証エラー

#### 問題: デバイス認証コードが期限切れになる

**原因:** `kiro login` 実行後にブラウザでの認証操作に時間がかかり、ユーザーコードの有効期限（通常5〜10分）が切れた。EC2ターミナルからWindowsブラウザへのURL・コードのコピーに手間取るケースが多い。

**解決方法:**

1. 現在のログインプロセスを `Ctrl + C` でキャンセルする

2. 再度 `kiro login` を実行する

   ```bash
   kiro login
   ```

3. 表示されたURLとコードを素早くWindows側のブラウザに入力する

> **Tips:** 事前にブラウザでAWS Builder IDにログインしておくと、認証完了までの時間を短縮できます。

**確認コマンド:**

```bash
kiro login
# 認証成功後に以下が表示されることを確認
# Successfully logged in!
```

#### 問題: ネットワーク接続エラーで認証が失敗する

**原因:** EC2インスタンスからKiro認証サーバー（`device.sso.us-east-1.amazonaws.com` 等）への HTTPS 通信がブロックされている。セキュリティグループのアウトバウンドルール、NATゲートウェイの設定不備、またはDNS解決の失敗が考えられる。

**解決方法:**

1. インターネットへの疎通を確認する

   ```bash
   curl -I https://aws.amazon.com
   ```

2. DNS解決が正常に動作しているか確認する

   ```bash
   nslookup device.sso.us-east-1.amazonaws.com
   ```

3. セキュリティグループのアウトバウンドルールを確認する
   - AWSコンソール → EC2 → セキュリティグループ → アウトバウンドルール
   - HTTPS（ポート443）へのアウトバウンド通信が許可されていることを確認

4. プライベートサブネットの場合、NATゲートウェイの状態を確認する
   - AWSコンソール → VPC → NATゲートウェイ → ステータスが「Available」であること

**確認コマンド:**

```bash
# DNS解決の確認
nslookup device.sso.us-east-1.amazonaws.com

# HTTPS接続の確認
curl -v https://device.sso.us-east-1.amazonaws.com 2>&1 | head -20

# 一般的な外部接続確認
curl -I https://registry.npmjs.org
```

#### 問題: 認証トークンが期限切れで操作が失敗する

**原因:** Kiro CLIの認証トークンには有効期限があり、一定期間が経過すると自動的に失効する。長期間使用していなかった場合や、セッションがタイムアウトした場合に発生する。

**解決方法:**

1. 再度ログインしてトークンをリフレッシュする

   ```bash
   kiro login
   ```

2. デバイス認証フローに従い、ブラウザで認証を完了する

3. 認証完了後、元の操作を再実行する

> **Tips:** 定期的に `kiro login` を実行してトークンを更新しておくと、突然の認証切れを防げます。

**確認コマンド:**

```bash
# 再認証後にKiro CLIが正常に動作することを確認
kiro login
kiro --version
```

---

### EC2リソース不足

#### 問題: メモリ不足（OOM: Out of Memory）でプロセスが強制終了される

**原因:** Node.jsプロジェクトのビルドやKiro CLIの実行時に大量のメモリを消費し、EC2インスタンスの物理メモリが不足した。t3.small（2 GiB RAM）の場合、大規模プロジェクトで発生しやすい。カーネルのOOM Killerによりプロセスが強制終了される。

**解決方法:**

1. 現在のメモリ使用状況を確認する

   ```bash
   free -h
   ```

2. OOM Killerのログを確認する

   ```bash
   sudo dmesg | grep -i "oom\|killed" | tail -10
   ```

3. スワップファイルを作成して一時的に対処する

   ```bash
   # 2GBのスワップファイルを作成
   sudo dd if=/dev/zero of=/swapfile bs=1M count=2048
   sudo chmod 600 /swapfile
   sudo mkswap /swapfile
   sudo swapon /swapfile

   # 永続化（再起動後も有効にする場合）
   echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
   ```

4. 長期的にはインスタンスタイプをt3.medium以上にアップグレードする

**確認コマンド:**

```bash
# メモリ使用状況の確認
free -h

# スワップが有効か確認
swapon --show

# OOM発生履歴の確認
sudo dmesg | grep -i "oom" | tail -5
```

#### 問題: ディスク容量が不足している

**原因:** `node_modules` ディレクトリ、npmキャッシュ、ログファイル等の蓄積によりディスクが逼迫した。特に複数プロジェクトを扱う場合や、EBSボリュームが20GB未満の場合に発生しやすい。

**解決方法:**

1. ディスク使用状況を確認する

   ```bash
   df -h /
   ```

2. 大きなディレクトリを特定する

   ```bash
   sudo du -sh /home/* /tmp/* /var/log/* 2>/dev/null | sort -rh | head -10
   ```

3. npmキャッシュをクリアする

   ```bash
   npm cache clean --force
   ```

4. 不要な `node_modules` を削除する

   ```bash
   # 特定のプロジェクトのnode_modulesを削除して再インストール
   rm -rf ~/projects/old-project/node_modules
   ```

5. ログファイルをローテーション・削除する

   ```bash
   sudo journalctl --vacuum-size=100M
   ```

6. 根本的にはEBSボリュームサイズを拡張する（AWSコンソールまたはCLI）

**確認コマンド:**

```bash
# ディスク使用率の確認
df -h /

# 現在のディレクトリごとの使用量確認
du -sh ~/projects/*/node_modules 2>/dev/null

# npmキャッシュサイズの確認
du -sh ~/.npm/_cacache 2>/dev/null
```

#### 問題: CPU使用率が高くパフォーマンスが低下する（CPUスロットリング）

**原因:** t3インスタンスのバースト可能なCPUクレジットが枯渇し、ベースラインパフォーマンス（t3.mediumで20%）に制限されている。長時間のビルド処理やテスト実行で発生しやすい。

**解決方法:**

1. CPU使用率とクレジット残高を確認する

   ```bash
   # CPU使用率の確認
   top -bn1 | head -5
   ```

2. AWSコンソールでCPUクレジットバランスを確認する
   - CloudWatch → メトリクス → EC2 → CPUCreditBalance

3. 一時的な対処: CPU集約型タスクの並列度を下げる

   ```bash
   # npmの並列インストールを制限
   npm install --maxsockets=2
   ```

4. 長期的な対処:
   - インスタンスタイプをt3.large以上にアップグレードする
   - `unlimited` モード（CPUクレジット無制限）を有効にする（追加課金あり）

   ```bash
   # AWS CLIでunlimitedモードを有効にする
   aws ec2 modify-instance-credit-specification \
     --instance-credit-specification "InstanceId=i-xxxxxxxxxxxxxxxxx,CpuCredits=unlimited"
   ```

**確認コマンド:**

```bash
# CPU使用率の確認
top -bn1 | grep "Cpu(s)"

# ロードアベレージの確認
uptime

# プロセスごとのCPU使用率
ps aux --sort=-%cpu | head -10
```

---

### SELinux・ファイアウォール関連

#### 問題: SELinuxがNode.jsやKiro CLIの実行をブロックする

**原因:** AlmaLinux 9ではSELinuxがデフォルトでEnforcingモードで動作しており、非標準のパスにインストールされたバイナリの実行や、特定のネットワーク接続をブロックする場合がある。特にnvm経由でインストールしたNode.jsがホームディレクトリ配下にある場合に発生しやすい。

**解決方法:**

1. SELinuxのステータスと拒否ログを確認する

   ```bash
   getenforce
   sudo ausearch -m AVC --start recent
   ```

2. 拒否されている操作を特定する

   ```bash
   sudo ausearch -m AVC -ts today | audit2why
   ```

3. 適切なSELinuxポリシーを追加する

   ```bash
   # 拒否ログからポリシーモジュールを生成
   sudo ausearch -m AVC -ts today | audit2allow -M kiro_node
   sudo semodule -i kiro_node.pp
   ```

4. 一時的な回避策（デバッグ目的のみ、本番環境では非推奨）

   ```bash
   # SELinuxをPermissiveモードに変更（拒否をログに記録するが実行は許可）
   sudo setenforce 0
   ```

   > **⚠️ 注意:** Permissiveモードは一時的なデバッグ目的のみ使用してください。問題が解決したら `sudo setenforce 1` でEnforcingモードに戻してください。

**確認コマンド:**

```bash
# SELinuxの現在のモード確認
getenforce

# SELinux拒否ログの確認
sudo ausearch -m AVC --start recent 2>/dev/null | head -20

# インストール済みカスタムモジュールの確認
sudo semodule -l | grep kiro
```

#### 問題: firewalldがアウトバウンド通信をブロックしている

**原因:** AlmaLinux 9のfirewalldがHTTPS（443）やSSH（22）のアウトバウンド通信をブロックしている。デフォルト設定ではアウトバウンドは許可されるが、カスタムゾーン設定やポリシーにより制限されている場合がある。

**解決方法:**

1. firewalldの状態と現在のゾーン設定を確認する

   ```bash
   sudo systemctl status firewalld
   sudo firewall-cmd --get-active-zones
   sudo firewall-cmd --list-all
   ```

2. 必要なサービス・ポートを許可する

   ```bash
   # HTTPSアウトバウンドを許可
   sudo firewall-cmd --permanent --add-service=https
   sudo firewall-cmd --permanent --add-service=http

   # SSHアウトバウンドを許可（GitLab接続用）
   sudo firewall-cmd --permanent --add-port=22/tcp

   # 設定を反映
   sudo firewall-cmd --reload
   ```

3. 特定のホストへの接続をテストする

   ```bash
   curl -I https://registry.npmjs.org
   ssh -T git@gitlab.com
   ```

**確認コマンド:**

```bash
# firewalldの状態確認
sudo firewall-cmd --state

# 許可されているサービス/ポートの一覧
sudo firewall-cmd --list-all

# 外部接続テスト
curl -sS -o /dev/null -w "%{http_code}" https://registry.npmjs.org
```

#### 問題: dnfリポジトリへの接続エラーが発生する

**原因:** AlmaLinux 9のdnfリポジトリやNodeSourceリポジトリへの接続が失敗している。リポジトリURLの変更、GPG鍵の期限切れ、ネットワーク設定の問題、またはリポジトリのメタデータキャッシュが破損している可能性がある。

**解決方法:**

1. dnfキャッシュをクリアして再構築する

   ```bash
   sudo dnf clean all
   sudo dnf makecache
   ```

2. リポジトリの設定ファイルを確認する

   ```bash
   ls /etc/yum.repos.d/
   sudo dnf repolist all
   ```

3. NodeSourceリポジトリが無効になっている場合は再追加する

   ```bash
   # 既存のNodeSourceリポジトリを削除
   sudo rm -f /etc/yum.repos.d/nodesource*.repo

   # 再度追加
   curl -fsSL https://rpm.nodesource.com/setup_20.x | sudo bash -
   ```

4. GPG鍵のエラーが出る場合はインポートする

   ```bash
   sudo rpm --import https://rpm.nodesource.com/gpgkey/nodesource-repo.gpg.key
   ```

5. DNS解決の問題がある場合は `/etc/resolv.conf` を確認する

   ```bash
   cat /etc/resolv.conf
   # 必要に応じてDNSサーバーを追加
   # nameserver 8.8.8.8
   ```

**確認コマンド:**

```bash
# リポジトリ一覧の確認
sudo dnf repolist

# メタデータの再取得
sudo dnf makecache

# パッケージ検索が正常に動作するか確認
dnf search nodejs
```


---

## 9. GitLab+Kiro統合ワークフロー

本セクションでは、Kiro CLIのspec駆動開発とGitLabのマージリクエスト（MR）ベースのコードレビューを組み合わせた、統合開発ワークフローを解説します。GitLabでのイシュー作成からMRのマージまでの一連の流れを、実際の開発シナリオに基づいてステップバイステップで説明します。

### エンドツーエンドフロー概要

Kiro CLIとGitLabを組み合わせた開発フローの全体像は以下の通りです。**GitLabイシューを起点**とし、featureブランチ上でspecを作成するのがポイントです。

```
┌─────────────────────────────────────────────────────────────────┐
│                   GitLab + Kiro 統合開発フロー                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. GitLabでイシュー作成                                          │
│     └── 機能要求・バグ報告をイシューとして起票                     │
│              ↓                                                   │
│  2. GitLabでfeatureブランチ作成                                   │
│     └── イシューからブランチを作成（GitLab UI）                    │
│              ↓                                                   │
│  3. ローカルでfeatureブランチにチェックアウト                       │
│     └── git fetch → git checkout feature/xxx                     │
│              ↓                                                   │
│  4. featureブランチ上でspec作成（kiro spec）                       │
│     └── requirements.md / design.md / tasks.md を記載           │
│              ↓                                                   │
│  5. タスク実装                                                    │
│     └── tasks.md のタスクを1つずつ実装                            │
│              ↓                                                   │
│  6. コミット（specタスク参照 + イシュー番号付き）                   │
│     └── コミットメッセージにspec/タスク情報とIssue #番号を含める    │
│              ↓                                                   │
│  7. プッシュ                                                      │
│     └── featureブランチをリモートにプッシュ                        │
│              ↓                                                   │
│  8. MR作成（イシューとspec情報を含む）                             │
│     └── MRのDescriptionにイシュー参照・spec参照・タスク一覧を記載  │
│              ↓                                                   │
│  9. コードレビュー                                                │
│     └── レビュアーがspecを参照しながらレビュー                     │
│              ↓                                                   │
│  10. マージ・クリーンアップ                                        │
│      └── マージ後にブランチを削除、イシューを自動クローズ          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

このフローにより以下のメリットが得られます：

| メリット | 説明 |
|---|---|
| **トレーサビリティ** | 要件→設計→実装→レビューまでの追跡が容易 |
| **レビュー品質の向上** | レビュアーがspecを参照し、実装が要件を満たしているか確認可能 |
| **タスク管理の一元化** | tasks.mdの進捗とGitのコミット履歴が連動 |
| **ナレッジ共有** | specがチーム内のドキュメントとして機能 |

### ブランチ戦略とKiro CLIの連携パターン

#### ブランチ命名規則

Kiro CLIのspecとGitLabのブランチを対応させるための命名規則です。

| パターン | 形式 | 用途 | 例 |
|---|---|---|---|
| spec対応ブランチ | `feature/<spec名>` | 1つのspecに対応する機能開発 | `feature/user-authentication` |
| バグ修正ブランチ | `fix/<spec名>` | バグ修正specに対応 | `fix/login-timeout-error` |
| ドキュメント更新 | `docs/<spec名>` | ドキュメント系specに対応 | `docs/api-reference-update` |

> **ポイント:** ブランチ名をspec名と一致させることで、ブランチとspecの対応関係が一目でわかります。

#### 1spec = 1ブランチの原則

Kiro CLIの1つのspecを1つのフィーチャーブランチで管理するパターンを推奨します。

```
.kiro/specs/
└── user-authentication/        ← spec
    ├── requirements.md
    ├── design.md
    └── tasks.md

git branch:
  main
  feature/user-authentication   ← 対応するブランチ
```

**この原則のメリット：**

- specのスコープとブランチのスコープが一致し、レビューしやすい
- MRの粒度がspecのタスク一覧と対応し、差分が明確になる
- マージ後のspec完了判定が容易

#### 大きなspecを分割する場合

specのタスクが多い場合は、サブブランチで段階的に開発することも可能です。

```
main
└── feature/user-authentication          ← メインのフィーチャーブランチ
    ├── feature/user-authentication/api  ← サブタスク用ブランチ
    └── feature/user-authentication/ui   ← サブタスク用ブランチ
```

この場合、サブブランチからフィーチャーブランチへのMR → フィーチャーブランチからmainへのMRという2段階でマージします。

#### specとMRの紐付けパターン

MRのDescriptionにspec情報を記載し、要件・設計とのトレーサビリティを確保します。

**MR Descriptionテンプレート：**

```markdown
## 概要
<!-- 変更内容の要約 -->

## 関連spec
- spec: `.kiro/specs/<spec名>/`
- requirements: `.kiro/specs/<spec名>/requirements.md`
- design: `.kiro/specs/<spec名>/design.md`

## 実装タスク
<!-- tasks.md から該当タスクをコピー -->
- [x] タスク1: ○○の実装
- [x] タスク2: △△の追加
- [ ] タスク3: □□のテスト（次のMRで対応）

## テスト結果
<!-- テストの実行結果 -->
- 単体テスト: PASS
- 結合テスト: PASS

## レビューのポイント
<!-- レビュアーに特に見てほしい箇所 -->
```

### 実践シナリオ: 機能開発のステップバイステップ

以下は、「ユーザー認証機能の追加」という機能リクエストを受けてから、MRのマージまでの一連のフローを示す実践例です。

#### ステップ1: GitLabでイシューを作成する

機能リクエストやバグ報告をGitLabのイシューとして起票します。

1. GitLabのプロジェクトページを開く
2. 「Issues」→「New issue」をクリック
3. 以下を入力：

| 項目 | 入力例 |
|---|---|
| **Title** | `ユーザー認証機能の実装` |
| **Description** | 要件の概要、背景、受入基準 |
| **Labels** | `feature`, `backend` |
| **Assignee** | 自分 |

4. 「Create issue」をクリック

> **Tips:** イシュー番号（例: `#42`）をメモしておきます。後のコミットメッセージやMRで参照します。

#### ステップ2: GitLabでfeatureブランチを作成する

イシューページから直接ブランチを作成します。

1. イシューの詳細ページで「Create branch」ボタンをクリック
2. ブランチ名を設定（例: `feature/user-authentication` または自動生成される `42-user-authentication`）
3. ソースブランチが `main` であることを確認
4. 「Create branch」をクリック

> **補足:** GitLabがイシュー番号付きのブランチ名を自動提案する場合もあります（例: `42-user-authentication`）。チームのルールに合わせてどちらでも構いません。

#### ステップ3: ローカルでfeatureブランチにチェックアウトする

EC2インスタンスのターミナルで、作成したブランチに切り替えます。

```bash
# プロジェクトルートに移動
cd ~/projects/your-project

# リモートの最新情報を取得
git fetch origin

# featureブランチにチェックアウト
git checkout feature/user-authentication
```

> **注意:** `git fetch` を忘れると、リモートで作成したブランチがローカルに反映されません。

#### ステップ4: featureブランチ上でspecを作成する

ブランチに切り替えた状態でspecを作成します。これにより、specファイルがfeatureブランチに含まれます。

```bash
# specを作成
kiro spec
```

`kiro spec` の対話プロンプトに従い、spec名（例: `user-authentication`）を設定します。

##### specファイルの初回コミット

specの骨格ができたら、早めにコミットしておきます。

```bash
git add .kiro/specs/user-authentication/
git commit -m "docs: user-authentication specを作成 #42

- requirements.md: 要件定義の骨格
- design.md: 設計ドキュメントの骨格
- tasks.md: タスク一覧の骨格

Refs: spec/user-authentication"
```

> **ポイント:** コミットメッセージに `#42`（イシュー番号）を含めることで、GitLab上でイシューとコミットが自動的にリンクされます。

#### ステップ5: requirements.mdの記載

VS Codeのエディタで `.kiro/specs/user-authentication/requirements.md` を開き、要件を定義します。

```markdown
# Requirements Document

## Introduction
ユーザー認証機能を実装し、ログイン・ログアウト・セッション管理を提供する。

## Requirements

### Requirement 1: ログイン機能
**User Story:** As a ユーザー, I want メールアドレスとパスワードでログインしたい...

#### Acceptance Criteria
1. THE System SHALL メールアドレスとパスワードによる認証を提供する
2. ...
```

#### ステップ6: design.mdの記載

要件に基づいてアーキテクチャ・設計を記載します。

```bash
# design.mdをエディタで編集
# .kiro/specs/user-authentication/design.md
```

#### ステップ7: tasks.mdの記載

設計に基づいて実装タスクを分解します。

```markdown
# Implementation Plan

## Tasks
- [ ] 1. 認証APIエンドポイントの作成
  - [ ] 1.1 POST /api/auth/login の実装
  - [ ] 1.2 POST /api/auth/logout の実装
- [ ] 2. セッション管理の実装
  - [ ] 2.1 JWTトークン生成ロジック
  - [ ] 2.2 ミドルウェアによるトークン検証
- [ ] 3. テストの作成
  - [ ] 3.1 認証APIの単体テスト
  - [ ] 3.2 セッション管理の結合テスト
```

spec（requirements → design → tasks）が揃ったらコミットします。

```bash
git add .kiro/specs/user-authentication/
git commit -m "docs: user-authentication specを完成 #42

- requirements.md: ログイン・ログアウト・セッション管理の要件定義
- design.md: 認証アーキテクチャの設計
- tasks.md: 実装タスクの分解

Refs: spec/user-authentication"
```

> **ポイント:** specファイルがfeatureブランチに含まれるため、MRでレビュアーが要件・設計を最初に確認できます。

#### ステップ8: タスクの実装とコミット

tasks.mdのタスクを1つずつ実装し、タスクごとにコミットします。

```bash
# --- タスク1.1: POST /api/auth/login の実装 ---

# コードを実装（VS Codeエディタで編集）
# ...

# タスク1.1完了時のコミット
git add src/api/auth/login.ts
git commit -m "feat: POST /api/auth/login エンドポイントを実装 #42

- メールアドレス/パスワードによる認証
- バリデーション処理の追加
- エラーレスポンスの定義

Refs: spec/user-authentication#1.1"

# tasks.mdのチェックボックスを更新
# - [x] 1.1 POST /api/auth/login の実装
git add .kiro/specs/user-authentication/tasks.md
git commit -m "docs: タスク1.1を完了としてマーク"
```

```bash
# --- タスク1.2: POST /api/auth/logout の実装 ---

# コードを実装...

git add src/api/auth/logout.ts
git commit -m "feat: POST /api/auth/logout エンドポイントを実装

- セッション無効化処理
- トークンブラックリスト追加

Refs: spec/user-authentication#1.2"
```

> **コミットメッセージのフォーマット:**
> ```
> <種別>: <変更の要約>
>
> - 詳細1
> - 詳細2
>
> Refs: spec/<spec名>#<タスク番号>
> ```
>
> `Refs:` 行でspecのタスク番号を参照することで、コミットとタスクの紐付けが明確になります。

#### ステップ9: テストの実行と確認

すべてのタスクを実装したら、テストを実行します。

```bash
# テストの実行
npm test

# ビルド確認
npm run build
```

テストが失敗した場合は修正してからコミットします。

#### ステップ10: リモートへのプッシュ

```bash
# フィーチャーブランチをリモートにプッシュ
git push -u origin feature/user-authentication
```

#### ステップ11: マージリクエスト（MR）の作成

GitLabでMRを作成します。MRのDescriptionにspec情報を含めることが重要です。

##### 方法A: GitLab Web UIから作成

1. GitLabのプロジェクトページを開く
2. 「Create merge request」バナーをクリック（またはMerge requests → New）
3. 以下の内容を入力：

**Title:**
```
feat: ユーザー認証機能の実装
```

**Description:**
```markdown
## 概要
ユーザー認証機能（ログイン・ログアウト・セッション管理）を実装しました。

Closes #42

## 関連spec
- 📋 spec: `.kiro/specs/user-authentication/`
- 📝 要件: `.kiro/specs/user-authentication/requirements.md`
- 🏗️ 設計: `.kiro/specs/user-authentication/design.md`

## 実装タスク
- [x] 1.1 POST /api/auth/login の実装
- [x] 1.2 POST /api/auth/logout の実装
- [x] 2.1 JWTトークン生成ロジック
- [x] 2.2 ミドルウェアによるトークン検証
- [x] 3.1 認証APIの単体テスト
- [x] 3.2 セッション管理の結合テスト

## テスト結果
- 単体テスト: ✅ PASS（12/12）
- 結合テスト: ✅ PASS（5/5）

## レビューのポイント
- JWTトークンの有効期限設定（design.mdのセキュリティ要件参照）
- パスワードハッシュ化アルゴリズムの選択
```

4. Assignee、Reviewer、Labelsを設定
5. 「Create merge request」をクリック

##### 方法B: GitLab CLI（glab）から作成

```bash
glab mr create \
  --title "feat: ユーザー認証機能の実装" \
  --description "## 概要
ユーザー認証機能を実装しました。

## 関連spec
- spec: .kiro/specs/user-authentication/
- 要件: .kiro/specs/user-authentication/requirements.md
- 設計: .kiro/specs/user-authentication/design.md

## 実装タスク
- [x] 1.1 POST /api/auth/login の実装
- [x] 1.2 POST /api/auth/logout の実装
- [x] 2.1 JWTトークン生成ロジック
- [x] 2.2 ミドルウェアによるトークン検証
- [x] 3.1 認証APIの単体テスト
- [x] 3.2 セッション管理の結合テスト" \
  --target-branch main \
  --assignee @me
```

#### ステップ12: コードレビュー（specを参照したレビュー）

レビュアーは以下の観点でレビューを行います。

| レビュー観点 | 確認方法 |
|---|---|
| **要件の充足** | `requirements.md` のAcceptance Criteriaがすべて実装されているか |
| **設計との整合性** | `design.md` のアーキテクチャに沿った実装か |
| **タスクの完了度** | `tasks.md` のすべてのタスクにチェックが入っているか |
| **コード品質** | 一般的なコードレビュー基準（可読性、保守性、テスト等） |

レビュアーはMRのDescriptionに記載されたspec参照リンクを使って、要件や設計ドキュメントを確認しながらレビューを進めることができます。

レビューでの指摘事項に対応する場合：

```bash
# レビュー指摘を修正
# ...

git add <修正ファイル>
git commit -m "fix: レビュー指摘対応 - トークン有効期限の修正

- JWTの有効期限をdesign.mdの仕様通り24時間に変更
- リフレッシュトークンの実装を追加

Refs: spec/user-authentication#2.1"

git push
```

#### ステップ13: マージとクリーンアップ

レビューが承認されたら、GitLab上でMRをマージします。

```bash
# マージ後、ローカルのブランチを整理
git checkout main
git pull origin main

# フィーチャーブランチを削除
git branch -d feature/user-authentication
```

tasks.mdのすべてのタスクが完了していることを最終確認します。

```bash
# specの完了状態を確認
cat .kiro/specs/user-authentication/tasks.md | grep "\- \["
```

**期待される出力（すべて完了）:**

```
- [x] 1.1 POST /api/auth/login の実装
- [x] 1.2 POST /api/auth/logout の実装
- [x] 2.1 JWTトークン生成ロジック
- [x] 2.2 ミドルウェアによるトークン検証
- [x] 3.1 認証APIの単体テスト
- [x] 3.2 セッション管理の結合テスト
```

> **Tips:**
> - マージ後もspecファイルはリポジトリに残り、機能のドキュメントとして参照できます
> - GitLabのMRにspec参照が含まれているため、後から実装の経緯を追跡する際にも役立ちます
> - 次の機能開発では、同じフロー（イシュー作成→ブランチ作成→spec作成→実装→MR）を繰り返します
> - MRのDescriptionに `Closes #番号` を含めると、マージ時にイシューが自動クローズされます

### specファイルの運用・管理方針

featureブランチごとにspecを作成してmainにマージしていくと、プロジェクトが成長するにつれて `.kiro/specs/` 配下にspecが蓄積されていきます。

```
.kiro/specs/
├── user-authentication/       ← MR #42でマージ済み
├── payment-integration/       ← MR #55でマージ済み
├── search-feature/            ← MR #68でマージ済み
├── bug-fix-login-timeout/     ← MR #73でマージ済み
└── dashboard-redesign/        ← 現在開発中
```

#### 運用パターン

チームの規模やプロジェクトの性質に応じて、以下のいずれかの方針を選択してください。

| 方針 | 内容 | 向いているチーム |
|---|---|---|
| **そのまま残す** | 完了済みspecもリポジトリに残す | 小〜中規模チーム、設計記録を重視 |
| **アーカイブする** | 完了済みspecを `_archived/` に移動 | 中〜大規模チーム、ディレクトリを整理したい |
| **削除する** | マージ後にspecを削除、MR履歴で追跡 | specよりコード中心のチーム |

#### 方針1: そのまま残す（推奨）

specを削除せずにリポジトリに残します。

**メリット：**
- 「なぜこう実装したか」の設計経緯をいつでも参照できる
- 新しいメンバーが過去の設計判断を理解しやすい
- grep/検索でspec内容を横断的に探せる

**デメリット：**
- ディレクトリが増え続ける（ただしビルドや実行には影響なし）

> **Tips:** `.kiro/specs/` はドキュメントのみなので、プロジェクトのビルドサイズやパフォーマンスには影響しません。

#### 方針2: アーカイブする

完了済みのspecを `_archived/` ディレクトリに移動して整理します。

```bash
# マージ完了後にアーカイブ
mkdir -p .kiro/specs/_archived
mv .kiro/specs/user-authentication .kiro/specs/_archived/

git add .kiro/specs/
git commit -m "chore: 完了済みspec(user-authentication)をアーカイブ"
```

アーカイブ後のディレクトリ構造：

```
.kiro/specs/
├── _archived/
│   ├── user-authentication/
│   ├── payment-integration/
│   └── search-feature/
└── dashboard-redesign/        ← アクティブなspecのみ表示
```

#### 方針3: 削除する（MR履歴で追跡）

specファイルをマージ後に削除し、GitLabのMR履歴で設計情報を追跡します。

```bash
# マージ完了後にspecを削除
rm -rf .kiro/specs/user-authentication/

git add .kiro/specs/
git commit -m "chore: 完了済みspec(user-authentication)を削除

MR !42 にspec内容が記録されているため、参照が必要な場合はMR履歴を確認してください。"
```

> **注意:** この方針を採用する場合は、MRのDescriptionにspec内容を十分に転記しておくことが重要です。

#### チームでの合意ポイント

specの運用方針はチームで統一してください。以下を決めておくとスムーズです。

- マージ後のspecの扱い（残す / アーカイブ / 削除）
- アーカイブのタイミング（マージ直後 / スプリント終了時 / 四半期ごと）
- specのブランチ命名とイシュー番号の紐付けルール

