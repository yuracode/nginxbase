# 第1回：Nginxインストールと静的ファイル配信

## 授業情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 90分（講義20分 + 演習60分 + まとめ10分） |
| 前提知識 | Linuxの基本コマンド、エディタ操作（nano/vim）、HTTPの基本 |
| 動作環境 | WSL2 (Ubuntu) |
| 到達目標 | 自分でconfファイルを書き、Nginxに静的HTMLページを返させる |

---

## 学習目標

この回が終わると、以下ができるようになります。

1. **Nginxをインストールして起動状態を確認できる**
2. **`/etc/nginx/` のディレクトリ構造の役割を説明できる**
3. **`server` ブロックと `location` ブロックを使った設定ファイルを書ける**
4. **`nginx -t` と `systemctl reload` を使って設定を安全に反映できる**

---

## 全体構成（最終形のうち、今回の範囲）

```
[ブラウザ]
    ↓ :80 (HTTP)
[Nginx on WSL2]    ← 今回はここまで
    ↓
[静的HTMLファイル]  ← 今回のターゲット
```

> 第2回でTomcat (Docker) を、第3回でTLS (HTTPS) を追加していきます。

---

## 1-1. Nginxインストール

```bash
sudo apt update
sudo apt install -y nginx
```

インストール確認：

```bash
nginx -v
```

起動：

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

> **確認ポイント**
> ブラウザで `http://localhost` にアクセスして「Welcome to nginx!」が表示されればOK。

---

## 1-2. ディレクトリ構造を把握する

```
/etc/nginx/
├── nginx.conf              ← メイン設定（基本触らない）
├── sites-available/        ← バーチャルホスト設定ファイルを置く場所
│   └── default             ← デフォルト設定（参考用）
├── sites-enabled/          ← sites-available へのシンボリックリンク
└── conf.d/                 ← 追加設定（今回はここを使う）
```

```bash
# デフォルト設定を見てみる
cat /etc/nginx/sites-available/default
```

ポイント：`server` ブロックと `location` ブロックの構造を確認する。

---

## 1-3. 静的ファイルを用意する

```bash
sudo mkdir -p /var/www/mysite
sudo nano /var/www/mysite/index.html
```

以下を記述して保存（`Ctrl+O` → `Enter` → `Ctrl+X`）：

```html
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>My Nginx Site</title>
</head>
<body>
  <h1>Nginxから配信しています</h1>
  <p>静的ファイル配信のテストです。</p>
</body>
</html>
```

---

## 1-4. バーチャルホスト設定を書く

```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```

以下を記述：

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 1-5. 設定を反映する

構文チェック：

```bash
sudo nginx -t
```

> `syntax is ok` / `test is successful` が出ればOK。
> エラーが出たら表示されたファイルと行番号を確認して修正する。

反映：

```bash
sudo systemctl reload nginx
```

---

## ゴール確認チェックリスト

- [ ] `nginx -v` でバージョンが表示される
- [ ] `systemctl status nginx` が `active (running)` になっている
- [ ] `sudo nginx -t` がエラーなく通る
- [ ] ブラウザで `http://localhost` にアクセスし、**「Nginxから配信しています」** が表示される
- [ ] `/etc/nginx/conf.d/mysite.conf` の中身を見ずに、自分の言葉で各ディレクティブを説明できる

---

## まとめ

| 概念 | 内容 |
|------|------|
| `server` ブロック | バーチャルホスト1つ分の設定単位 |
| `listen` | 受け付けるポート番号 |
| `root` | 静的ファイルのルートディレクトリ |
| `location` | URLパスごとの処理ルール |
| `nginx -t` | 設定ファイルの構文チェック |
| `systemctl reload` | サービスを止めずに設定を再読み込み |

---

## 発展課題（任意）

1. `index.html` に加えて `/about.html` を作り、`http://localhost/about.html` で表示できることを確認する。
2. `/etc/nginx/conf.d/mysite.conf` の `server_name` を `mysite.local` に変更し、`/etc/hosts` に `127.0.0.1 mysite.local` を追記して、名前ベースのアクセスを試す。
3. `access_log` と `error_log` のパスを調べ、ブラウザでアクセスしたときにログが追記されることを `tail -f` で観察する。

---

## 次回予告

第2回では、**Docker上でTomcatを起動**し、Nginxを**リバースプロキシ**として使ってJSPページを返す構成を作ります。
今回の `mysite.conf` を編集して、静的配信からプロキシ転送に切り替えていきます。
