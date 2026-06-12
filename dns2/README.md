# DNS名前解決エラー デバッグ ハンズオン

DNS名前解決が失敗する代表的なケース（NXDOMAIN / SERVFAIL）を再現し、dig でデバッグする方法を学ぶハンズオン教材。

## ディレクトリ構成

```
configs/    CoreDNS設定ファイル（障害シナリオ別）
handson/    ハンズオン手順書
slides/     MARPスライド
```

## 必要なもの

- macOS (dig は標準搭載)
- Docker Desktop for Mac
- MARP CLI (`npm install -g @marp-team/marp-cli`)

## 使い方

```bash
# 障害環境を起動
docker compose up -d

# 動作確認
dig @127.0.0.1 -p 15301 www.example.test  # → NOERROR（正常）
dig @127.0.0.1 -p 15301 app.example.test  # → NXDOMAIN（レコード不在）
dig @127.0.0.1 -p 15302 www.google.com    # → SERVFAIL（上流障害）

# 後片付け
docker compose down
```

## スライドのビルド

```bash
marp slides/dns_debugging.md --html
```
