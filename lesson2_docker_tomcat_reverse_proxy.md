# 第2回：Dockerコンテナ上のTomcatとリバースプロキシ

## 授業情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 90分（講義25分 + 演習55分 + まとめ10分） |
| 前提 | 第1回が完了していること（Nginxが起動し、`mysite.conf` で `http://localhost` が応答する状態） |
| 追加要件 | Docker / Docker Compose がインストール済みであること |
| 到達目標 | DockerでTomcatを起動し、NginxをリバースプロキシとしてJSPページを返させる |

---

## 学習目標

1. **Webサーバー（Nginx）とアプリサーバー（Tomcat）の役割の違いを説明できる**
2. **Dockerfile と docker-compose.yml を書いて Tomcat コンテナを起動できる**
3. **`proxy_pass` を使ったリバースプロキシ設定が書ける**
4. **`proxy_set_header` で何を伝えているかを説明できる**

---

## 今回作る構成

```
[ブラウザ]
    ↓ :80
[Nginx on WSL2]
    ↓ proxy_pass http://localhost:8080
[Tomcat on Docker :8080]
    ↓
[hello.jsp]
```

> 第1回で書いた `mysite.conf` を、静的配信からリバースプロキシ構成へ書き換えます。

---

## 事前確認

```bash
docker --version
docker compose version
```

> どちらもバージョンが返ってくればOK。
> 返ってこない場合は Docker Desktop（WSL2 Integration有効）または Docker Engine をセットアップしてください。

---

## 2-1. アプリファイルを準備する

作業ディレクトリを作成：

```bash
mkdir -p ~/tomcat-app
cd ~/tomcat-app
```

JSPファイルを作成：

```bash
nano hello.jsp
```

以下を記述：

```jsp
<%@ page contentType="text/html; charset=UTF-8" %>
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>Hello Tomcat</title>
</head>
<body>
  <h1>Hello from Tomcat!</h1>
  <p>サーバー時刻: <%= new java.util.Date() %></p>
  <p>リクエストURI: <%= request.getRequestURI() %></p>
</body>
</html>
```

---

## 2-2. Dockerfileを作成する

```bash
nano Dockerfile
```

```dockerfile
FROM tomcat:10-jdk17
# デフォルトのROOTアプリを削除してクリーンにする
RUN rm -rf /usr/local/tomcat/webapps/ROOT
# ROOTアプリとしてhello.jspを配置
RUN mkdir -p /usr/local/tomcat/webapps/ROOT
COPY hello.jsp /usr/local/tomcat/webapps/ROOT/
```

---

## 2-3. docker-compose.ymlを作成する

```bash
nano docker-compose.yml
```

```yaml
services:
  tomcat:
    build: .
    ports:
      - "8080:8080"
    restart: unless-stopped
```

---

## 2-4. Tomcatを起動する

```bash
docker compose up -d
```

起動確認：

```bash
docker compose ps
docker compose logs tomcat
```

> **確認ポイント**
> ブラウザで `http://localhost:8080` にアクセスして
> 「Hello from Tomcat!」が表示されればOK。

---

## 2-5. Nginxのリバースプロキシ設定を追加する

第1回で作成した設定ファイルを編集：

```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```

以下のように書き換える：

```nginx
server {
    listen 80;
    server_name localhost;

    # 静的ファイルはNginxが直接返す
    location /static/ {
        root /var/www/mysite;
    }

    # それ以外はTomcatへ転送（リバースプロキシ）
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## 2-6. 設定を反映する

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## ゴール確認チェックリスト

| アクセス先 | 期待する結果 |
|------------|-------------|
| `http://localhost:8080` | Tomcatに直接アクセスできる（Hello from Tomcat!） |
| `http://localhost` | Nginx経由で同じJSPページが表示される |
| `docker compose ps` | `tomcat` サービスが `Up` 状態 |
| `sudo nginx -t` | エラーなく通る |

> **ポイント**
> 最終的に本番環境では8080を外部公開しません。
> 80/443だけ公開し、Nginxが内部でTomcatに転送するのが正しい構成です。

---

## まとめ

| 概念 | 内容 |
|------|------|
| リバースプロキシ | クライアントの代わりにNginxがアプリサーバーへリクエストを転送 |
| `proxy_pass` | 転送先URLを指定するディレクティブ |
| `proxy_set_header` | 転送時に付加するHTTPヘッダー（クライアントIPなどを伝える） |
| Webサーバーの役割 | 静的ファイル配信・SSL終端・ロードバランシング |
| アプリサーバーの役割 | ビジネスロジック・動的コンテンツ生成 |

---

## 発展課題（任意）

1. `/static/` 配下に画像ファイルを置き、JSPページから `<img src="/static/xxx.png">` で参照して、**静的はNginx・動的はTomcat**の役割分担が機能していることを確認する。
2. `docker compose logs -f tomcat` を流しながら `http://localhost` にアクセスし、Nginx経由のリクエストでもTomcatログにアクセスが記録されることを観察する。
3. JSPで `request.getHeader("X-Real-IP")` と `request.getRemoteAddr()` を表示し、`proxy_set_header` の効果を比較する。

---

## トラブルシューティング

| 症状 | 確認ポイント |
|------|-------------|
| `502 Bad Gateway` | Tomcatコンテナが起動しているか（`docker compose ps`） |
| `8080` だけ表示できない | ポート競合 / `docker compose logs` でエラー確認 |
| `nginx -t` でエラー | `;` の付け忘れ・`{}` の閉じ忘れが多い |

---

## 次回予告

第3回では、**TLS（HTTPS化）**を行います。
自己署名証明書を作成し、`https://localhost` でアクセスできるようにし、HTTP→HTTPSの自動リダイレクトも設定します。
今回の `mysite.conf` をさらに書き換えて、80番ポートはリダイレクト専用、443番ポートが本来の入口、という構成にします。
