# Vault Enterpriseについて（１）

こんにちは。HashiCorp Japanの伊藤です。HashiCorpには４つのEnterprise製品があります。以下がそれぞれのプロダクトのOSS vs enterpriseの比較資料になります。

1. [Terraform Etnerprise](https://www.hashicorp.com/products/terraform/offerings)
2. [Vault Enterprise](https://www.hashicorp.com/products/vault/enterprise)
3. [Consul Enterprise](https://www.hashicorp.com/products/consul/enterprise)
4. [Nomad Enterprise](https://www.hashicorp.com/products/nomad#features)

今回はこの中の**Vault**についての説明をしたいと思います。(はじめに謝っておきます。Enterpriseの機能紹介まで書きたかったのですが、Vaultの基本概念だけで結構なボリュームになったので、Enterprise版については次回書きます・・・)

---

## Vaultとは？

まずは**Valut**という製品について、ざっくりと説明させていただきます。

### Vaultは何をしてくれるの？

Vaultは従来のStaticなインフラからDynamicなインフラへ切り替わっている昨今において、従来のセキュリティモデルでは対応できない、もしくは対応が難しいシークレットの扱いを容易に且つセキュアに解決するソリューションツールです。s

シークレットと言うと非常に広義なモノになるのですが、例えば：
* パスワード
* データベースへのアクセスクレデンシャル
* クラウドへアクセスする際のクレデンシャル
* SSHキー
* PKIの証明書
* などなど

があります。

まず、最初に言いたい事として、**Valutが扱えるシークレットは多岐にわたる**ということです。ですので、Vaultのシークレット管理機能を全て説明するのは、とてもとても一つの記事では書きつくせません。この点については、追い追い少しづつ紹介していきたいと思っていますが、なにせVaultは開発も活発で、どんどん新しい機能が追加されています。

とはいえ、Vaultの究極の目的を一言で言ってしまえば、**ユーザーもしくはアプリケーションに対して、それが必要とするシークレット** を**アクセスコントロール**に基づいて提供する事、となります。つまり、**「誰が」「何に」「アクセス」**できるのか、を制御し、**「誰が」「何を」「したか？」**を記録することです。

### Vaultの使い方

Vaultを利用するには以下の３つの方法があります。

1. CLI (Command line interface)
  * コマンドラインツールによるVaultとのやり取り
  * ある程度抽象化されているので学習するのが容易
  * コマンドラインツールなのでスクリプトなどに組み込むのが容易
  * ほとんどのVault上の処理を行える（全てではない)


2. GUI (Graphical user interface)
  * ブラウザでVaultのUIにアクセス（defaultではPort 8200)
  * Vaultバイナリがインストールされてない環境からでもVaultの状況チェックや、ある程度の設定などが可能
  * 人間向け
  * 人間にとって使いやすい反面、できる事は限定的
    * 視覚化されているのでデモなどに利用しやすい。（私個人的に、お客様へ説明しやすいです）


3. API (Application appliation interface)
  * WEB API / RESTfull API
  * Vault上の全ての処理がAPIで可能（言ってしまえば、CLIもGUIもAPIの一部の機能へのラッパーに過ぎない）
  * APIなので、アプリケーションに組み込みやすい



### Vaultの基本概念について


Vaultの基本概念を私なりにざっくりと項目にしてみました。

1. Seal/Unseal
2. Authentication Methods
3. Secret Engine
4. Policy
5. Audit
6. Telemetry

一つずつ簡単に説明しますと、

1. Seal/Unseal

  * Vaultは全てのシークレットをストレージ上に**暗号化**して保管します。そのため、**暗号化**をするための**暗号キー**が必要になります。そしてその暗号キーは別の**マスターキー**というもので暗号化されています。（すでにややこしいですね）。その**マスターキー**は、基本的にどこにも保管されない仕組みになっており、Vaultは初期化時や再起動時はマスターキーが分からないため何も出来ない状態になっています。そのマスターキーを復元する事を**Unseal**と言います。
  * **Unseal**にはいくつかの方法があります。詳細は省きますが、以下の方法があります。
    * マスターキー をいくつかのキーに分割し、複数の場所に保管する
      * ５つのキーに分割し、そのうち３つがあれば復元できる、など。これは[シャミアの秘密分散法](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing)というものを使っています。
      * 例えば、５つのキーを５人のセキュリティ管理者に振り分けることで、個々人によるUnsealを防ぐ、など。
    * HSM(Hardware Security Module)にマスターキーを保管する
      * HSM（Hardware Security Module)は物理的なデバイスです。
      * 現時点ではEnterprise版でしかサポートしていません。
    * Cloud HSMにマスターキーを保管する
      * AWS KMS, Google Cloud KMS, Azure Key Vaultなど
      * より詳細については、[こちら](https://www.hashicorp.com/blog/enabling-cloud-based-auto-unseal-in-vault-open-source)を参照ください。
    * 別のVault上にマスターキー を保管する（Vault v1.2以上でサポート）
      * 別のVaultサーバーにマスターキーを保管する方法です。Transit auto-unsealという機能になります）
  * いざ、何かしらの不穏なアクティビティがあった場合に、VaultをSealする事で、それ以上のシークレット提供を強制的に停止する、という事が出来ます。


2. Authentication Methods

  * **「誰が」「何に」「アクセスできる」** という文面でいうところの**「誰が」**の部分になります。
  * 多くのケースでは、**セーフリストにあるIPアドレス**などで判別していたと思います。これらは従来のFirewallに守られた城壁内で、また個別に特有のPrivate IPが割り振られている世界では比較的安全であると考えられます**(High-trust network)**。
  * ただ、これからのマイクロサービスアーキテクチャに代表される世界では、様々な環境で様々なVMであったりコンテナであったりが多様に入り混じることとなります**(Low-trust network)**。
  * よって、従来の**IPベースの判別**から**Identityベースの判別**のようなものが必要となります。
  * そこでVaultは**信頼できる認証基盤**により、**認証（Authentication)**を行います。VaultはVault自体の認証方式（AppRoleやUsername+Passwordなど）に加え、様々な外部認証基盤をを利用できます。
    * LDAP, Cloud認証基盤（IAMなど）, OIDC, Okta, TLSなど
    * サポートしている認証基盤については、[こちら](https://www.vaultproject.io/docs/auth/)
  *   * プラグイン式ですので、カスタムAuthentication Methodを作成することも可能です。
      * https://www.vaultproject.io/docs/plugin/index.html



3. Secret Engine

  * **「誰が」「何に」「アクセス」**できる、という文面でいうところの**「何に」**に当たる部分になります。何っていうのは、つまるところシークレットです。
  * VaultはK/V(Key-Value)のような静的なシークレットを安全に保管し、また、それを安全に別のIdentityへ渡すメカニズムを持っています（詳細については、またいつか書きます）
    * 有効な**静的なユーザーネーム＋パスワード**や**静的なAPIトークン**など
  * ただ、Vaultは静的なシークレットの管理以外に、**動的なシークレット**の生成において最も強みを発揮します。
    * 例えば：
      * 1分間だけ有効なPostgreSQLへのユーザーネームを発行したい
      * 1時間だけ有効なAWSのIAMキーを発行したい
      * 1週間だけ有効なPKIのクライアント証明書を発行したい
      * 1回だけ有効なSSHのパスワードを発行したい（One-time password)
    * 発行した動的シークレットを状況に応じて**無効化（Revoke）**する機能があります。
      * データベースへInsertやSelectなど必要な処理を行ったら即座にRevokeする。
      * AWSやGCP上で必要な処理を行ったら即座にRevokeする。
    * これにより、もし外部にシークレットが漏れてしまったとしても、即座にRevokeすることで被害を最小限に留める事ができます。
    * また全ての動的シークレットにはTTL(Time-to-live)が設定されていますので、放置していたとしてもいずれは無効化されます。
  * Secret Engineで扱えるシークレットは先に述べた以下のようなものがあります。
    * パスワード
    * データベースへのアクセスクレデンシャル
    * クラウドへアクセスする際のクレデンシャル
    * SSHキー
    * PKIの証明書
    * などなど
  * 現時点でSecret Engineのリストは[こちら](https://www.vaultproject.io/docs/secrets/)
  * プラグイン式ですので、カスタムSecret Engineを作成することも可能です。
    * https://www.vaultproject.io/docs/plugin/index.html


4. Policy

  * **「誰が」「何に」「アクセス」**できる、という文面でいうところの**「アクセスできる」**に当たる部分になります。アクセスコントロールのことです。
  * 2のAuthentication Methodsで認証されたIdentityへの**認可**(何をしていいか、の明記)
  * APIのPathへのアクセス制限
    * Vault上の全てのシークレット（静的・動的共に）および管理系のAPIは全て*Path*で管理されます。またそれがそのままAPIのエンドポイントになります。例えば、
      * K/Vエンジンのmy-secretというシークレットを読む場合
        * [API] curl https://127.0.0.1:8200/v1/secret/my-secret
        * [CLI] vault read secret/my-secrets
      * VaultをSealする（＊要管理者権限）
          * [API] curl --request PUT http://127.0.0.1:8200/v1/sys/seal
          * [CLI] vault operator seal
    * 例：
      * `secret/my-secret`へのアクセスコントロール(CRUD＋List)
        *```
path "secret/my-secret"  
{
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

      * Sealする権限を与える(＊要管理者権限)
        *```
path "sys/seal"
{
  "sudo"
}
```

  * おそらく、運用者側からすると一番念を入れて作成する者になると思います。

5. Audit

  * Vaultを運用する上で、**「誰が」「何を」「したか？」**を実際に記録する機能になります。
    * 全てのAPIのログを記録します。**誰がいつ何にどのようなアクセスして、どのような結果になったか？｀**が全て記録されます。
    * Auditの記録先としては以下をサポートしています。詳細は[こちら](https://www.vaultproject.io/docs/audit/)
      * File
      * Syslog
      * Socket
    * Audit機能により、何かあった場合のトラッキングが可能になり、規制の厳しい業界の制定（金融業界など）にも対応可能となります。


6. Telemetry

  * Vaultのあらかたの機能などは理解いただけたと思いますが、結局のところ最も重要なのはProduction環境での運用です。今この時点でVaultがどういう状態なのか、なにか問題が怒っていないか、リソースは十分に行き渡っているか、などを**監視（Monitoring）**する必要があります。
    * 例えば、VaultのCPU稼働率、メモリ使用量、APIリクエストへの応答時間、クラスタ内の状況、などなど
  * Vaultには実稼働状態の各種メトリクスを出力する**Telemetry**という機能があります。
    * 対応している監視ツールは、statsd, circonus, dogstatsd, prometheus, などがございます。
    * 詳細については[こちら](https://www.vaultproject.io/docs/configuration/telemetry.html)
    * また、Vaultが計測可能なメトリクス一覧は[こちら](https://www.vaultproject.io/docs/internals/telemetry.html)
  * この機能により、運用状況を各種ツールでモニタリングし、何か問題が起きそうな場合にアラートをあげたり、問題が起きた場合に即座に対応する事が可能となります。

---
## まとめ

長い文章となりましたが、HashiCorpのVaultという製品の概要を記載させていただきました。まとめますと：

**Vaultは「誰が」「何に」「アクセス**できるかを制御し、**「誰が」「何を」「したか」**を全て記録するツールです。

今回のブログでは、これらVaultの内部的な働きについての説明であり、技術的な部分に過ぎません。今回は概要説明だけでボリュームを使い果たしてしまいましたが、本当に記載したかったものは、**Vaultがビジネスにおいてどう役にたつか？**という部分です。これについては、次回がっつり書きます。

おたのしみあれ。
