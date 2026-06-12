# DNS名前解決エラー デバッグ ハンズオン — 実装プラン

## Context

ネットワーク勉強会（Mastering TCP/IP 勉強会）の教材。DNS名前解決が失敗する2つの代表的ケースを再現し、参加者（アプリ開発者）が dig でデバッグする方法をハンズオンで学ぶ。**30分のコンパクトなセッション**。

## 決定事項

| 項目 | 決定 |
|------|------|
| 時間枠 | 30分（スライド5分 + ハンズオン20分 + まとめ5分） |
| 対象者 | アプリ開発者（dig 未経験者も想定） |
| シナリオ | 2つ: NXDOMAIN + SERVFAIL |
| DNSサーバー | CoreDNS (Docker) |
| デバッグ範囲 | DNSプロトコル層のみ（dig コマンド中心） |
| スクリプト | 不要（dig だけで完結） |
| スライド | MARP形式で作成 |

## アプローチ

- **Docker Compose** で CoreDNS コンテナを2つ起動（NXDOMAIN用 + SERVFAIL用）
- **デバッグはホストの macOS から** `dig @127.0.0.1 -p <port>` で実行
- ホストの DNS 設定は一切変更しない
- ゾーン名は `example.test`（RFC 6761 予約済み TLD）
- ポート番号: 15301 (NXDOMAIN), 15302 (SERVFAIL)

## ディレクトリ構成

```
dns/
├── README.md
├── .gitignore
├── compose.yaml
├── slides/
│   └── dns_debugging.md          # MARPスライド
├── handson/
│   └── handson_guide.md          # ハンズオン手順書
└── configs/
    ├── nxdomain/
    │   ├── Corefile
    │   └── db.example.test
    └── servfail/
        └── Corefile
```

## 障害シナリオ

### Scenario 1: NXDOMAIN（レコード不在）

- **ポート**: 15301
- **仕組み**: CoreDNS が `example.test` ゾーンを権威的に提供。`www` と `api` の A レコードはあるが `app` は意図的に欠落
- **期待される症状**: `dig @127.0.0.1 -p 15301 app.example.test` → `status: NXDOMAIN`
- **デバッグの学び**:
  - NXDOMAIN = 「その名前はこのゾーンに存在しない」
  - 正常なレコード (`www`) と比較して問題を切り分ける
  - SOA レコードの MINIMUM フィールド = ネガティブキャッシュ TTL

### Scenario 2: SERVFAIL（上流障害）

- **ポート**: 15302
- **仕組み**: CoreDNS が全クエリを到達不能IP (192.0.2.99) にフォワード
- **期待される症状**: `dig @127.0.0.1 -p 15302 www.google.com` → `status: SERVFAIL`
- **デバッグの学び**:
  - SERVFAIL = 「サーバーが処理中にエラーが発生した」
  - 別の DNS サーバー (`dig @8.8.8.8`) で同じクエリを試して問題を切り分ける
  - `docker logs` でサーバー側のエラーを確認する方法

## CoreDNS 設定

### configs/nxdomain/Corefile
```
example.test:53 {
    file /etc/coredns/db.example.test
    log
    errors
}
```

### configs/nxdomain/db.example.test
```
$ORIGIN example.test.
@   IN SOA ns1.example.test. admin.example.test. (
        2024010101 3600 900 86400 300 )
    IN NS  ns1.example.test.
ns1 IN A   127.0.0.1
www IN A   192.0.2.1
api IN A   192.0.2.2
; app は意図的に欠落 → NXDOMAIN
```

### configs/servfail/Corefile
```
.:53 {
    forward . 192.0.2.99
    log
    errors
}
```

## compose.yaml

```yaml
services:
  dns-nxdomain:
    image: coredns/coredns:1.12.0
    container_name: dns-nxdomain
    ports:
      - "15301:53/udp"
      - "15301:53/tcp"
    volumes:
      - ./configs/nxdomain/Corefile:/etc/coredns/Corefile
      - ./configs/nxdomain/db.example.test:/etc/coredns/db.example.test
    command: ["-conf", "/etc/coredns/Corefile"]

  dns-servfail:
    image: coredns/coredns:1.12.0
    container_name: dns-servfail
    ports:
      - "15302:53/udp"
      - "15302:53/tcp"
    volumes:
      - ./configs/servfail/Corefile:/etc/coredns/Corefile
    command: ["-conf", "/etc/coredns/Corefile"]
```

## ハンズオン手順書の構成

ip_addressing の「観察ポイント → コマンド → 答え合わせ → 解説」フォーマットに準拠:

1. **事前準備** (2分): `docker compose up -d`, `dig` の基本的な使い方
2. **Step 1: 正常系ベースライン** (3分): `dig @8.8.8.8 www.google.com` で正常な応答を確認、dig 出力の読み方
3. **Step 2: NXDOMAIN** (8min): 存在するレコード vs 存在しないレコードの比較
4. **Step 3: SERVFAIL** (8min): 上流障害の切り分け、docker logs での確認
5. **まとめ: 切り分けチェックリスト** (3min): NXDOMAIN vs SERVFAIL の違い、デバッグ手順

## MARP スライドの構成

5分で収まるミニマルなスライド:

1. タイトル + 今日のゴール
2. DNS名前解決の基本フロー（1枚の図）
3. dig コマンドの読み方（status / flags / ANSWER セクション）
4. エラーコード一覧表（NOERROR / NXDOMAIN / SERVFAIL / REFUSED）
5. 「ハンズオンで体験しましょう」

## 実装順序

1. `.gitignore`, `README.md`
2. `configs/` 配下の Corefile とゾーンファイル
3. `compose.yaml`
4. `handson/handson_guide.md`
5. `slides/dns_debugging.md`
6. 動作検証: `docker compose up -d` → dig で各シナリオ確認
