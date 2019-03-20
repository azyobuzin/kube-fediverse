# kube-fediverse
Kubernetes でつくる隔離 ActivityPub ネットワーク

- [Kubernetesで隔離Mastodonネットワークを作った - アジョブジ星通信](https://azyobuzin.hatenablog.com/entry/2019/03/21/024504)

## このリポジトリは何？
通信が外に漏れださないように Network Policy を設定した、 Mastodon コンテナを動かすための Kubernetes リソースの YAML 群です。

## TODO
- [ ] Helm 化
- [ ] Mastodon 以外のサーバー

## 構築方法
### 前提条件
- 新しめの Kubernetes クラスタ（Kubernetes v1.13 以外はわかりません）
- Network Policy が使えるプラグインがインストールされていること
- Ingress Controller がインストールされていること
- `fediverse` 名前空間が存在すること（`01-namespace.yaml`）

### 1. ストレージクラスの準備
ストレージクラスとして、 PostgreSQL を動かすための高速なストレージクラス `fediverse-database` と、画像アップロード機能の保存先として使う `fediverse-appdata` を用意します。 `name` がこの通りであれば、何でも構いません。

例として、 `02-storage` ディレクトリに、 `fediverse-database` を [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)、 `fediverse-appdata` を [nfs-provisioner](https://github.com/kubernetes-incubator/external-storage/tree/nfs-provisioner-v2.2.1-k8s1.12/nfs) でダイナミックプロビジョニングする構成を用意してあります。

Local Path Provisioner では、 Pod が割り当てられたノードの `/var/local/fediverse/local-path` にデータが保存されます。一度ノードが決まった Pod は、 PersistentVolumeClaim を削除しない限り、そのノードにのみスケジュールされます。

nfs-provisioner については、まず、 NFS サーバーとして使うノードを決め、そのノードに `fediverse/nfs: "true"` というラベルを設定してください。その後、 nfs-provisioner をデプロイすると、そのノードの `/var/local/fediverse/nfs` にデータを保存します。

### 2. TLS 証明書の作成
HTTPS を使用するために証明書を用意します。この証明書は、コンテナに自動的にインストールされるので、適当な自己署名証明書で大丈夫です。

作成した証明書を `fediverse-cert` という名前で、 Kubernetes に登録します。

```
kubectl create secret tls -n fediverse fediverse-cert --cert=path/to/cert/file --key=path/to/key/file
```

### 3. 設定とデータベースのデプロイ
最初の Mastodon サーバーとして「mastodon1」（mastodon1.fediverse.local.azyobuzi.net）を立ち上げます。

`mastodon1` ディレクトリの `01-config.yaml`、`02-networkpolicies.yaml`、`03-database.yaml` を apply してください。これで設定類とデータベース（PostgreSQL、Redis）をデプロイできました。データベースの Pod が起動したことを確認して、次に進んでください。

### 4. データベースのセットアップ
Mastodon のデータベースを初期化します。これは `rake db:migrate` コマンドを実行することになります。このコマンドの呼び出しを Job として用意したものが、 `04-setupjob.yaml` ですので、これを実行し、完了するまで待ってください。

### 5. Mastodon のデプロイ
Mastodon サーバーと、 Sidekiq ワーカーをデプロイし、 Ingress を使用してアクセスできるようにします。 `05-mastodon.yaml` と `06-ingress.yaml` を apply することで、完了します。

### 6. DNS の設定
クラスタネットワーク内から「*.fediverse.local.azyobuzi.net」を解決しようとすると、 Ingress Controller のアドレスを返すように設定します。

例えば、 DNS サーバーとして CoreDNS、 Ingress Controller として Traefik を使用している場合、 CoreDNS の設定の、 Kubernetes プラグインの前に、次の設定を追加します。

```
template IN A {
    match ^(.*\.)?fediverse\.local\.azyobuzi\.net\.$
    answer "{{ .Name }} 60 IN CNAME traefik.kube-system.svc.cluster.local."
    upstream
    fallthrough
}
```

また、クラスタ外からアクセスできるように、ホストマシンが使用している DNS を設定するか、 hosts ファイルを編集して、 *.fediverse.local.azyobuzi.net への通信を、クラスタノードに向くように設定します。

### 7. 2つめの Mastodon のデプロイ
「mastodon2」を作成したい場合、 `mastodon1` ディレクトリをコピーし、ディレクトリ内に含まれる全 YAML ファイルの「mastodon1」を「mastodon2」に書き換え、同様の手順でデプロイしてください。

### 8. 通信を監視する
サーバーの「/rin/」にアクセスすると、そのサーバーへのリクエストを見ることができます。
