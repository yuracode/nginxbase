# 第3回：TLS（HTTPS化）

## 授業情報

| 項目 | 内容 |
|------|------|
| 所要時間 | 90分（講義25分 + 演習55分 + まとめ・振り返り10分） |
| 前提 | 第2回が完了していること（`http://localhost` でNginx経由でTomcatのJSPが表示される状態） |
| 到達目標 | 自己署名証明書でNginxをHTTPS化し、HTTP→HTTPSリダイレクトを構成する |

---

## 学習目標

1. **TLSハンドシェイクの流れ（公開鍵・共通鍵の役割）を説明できる**
2. **`openssl req` で自己署名証明書を生成できる**
3. **NginxでHTTPS（443番ポート）を有効にできる**
4. **HTTP→HTTPSリダイレクトの仕組みを構成できる**
5. **本番環境との違い（Let's Encrypt等）を説明できる**

---

## 今回完成させる構成

```
[ブラウザ]
    |
    | :80 → 301リダイレクト
    | :443 (TLS終端)
    |
[Nginx on WSL2]
    |  証明書検証・暗号化・復号
    |  proxy_pass（内部はHTTP）
    ↓
[Tomcat on Docker :8080]
    ↓
[hello.jsp]
```

---

## 3-1. TLS/HTTPSの仕組み（概念整理）

```
[クライアント]                    [サーバー]
     |                               |
     |── TLSハンドシェイク ──────────>|
     |<── サーバー証明書（公開鍵）────|
     |── 共通鍵を暗号化して送信 ────>|
     |              （ここから暗号化通信）
     |<────── 暗号化されたHTTPレスポンス ─|
```

| 用語 | 意味 |
|------|------|
| 証明書（.crt） | サーバーの公開鍵 + 所有者情報。CAが署名する |
| 秘密鍵（.key） | サーバーだけが持つ鍵。絶対に外部公開しない |
| 自己署名証明書 | CAではなく自分で署名した証明書。開発・学習用 |
| CA（認証局） | 証明書に署名して信頼性を保証する第三者機関 |

> ブラウザの「この接続は安全ではありません」警告は、
> 証明書が信頼されたCAによって署名されていないために出ます。
> 自己署名証明書は **開発・学習専用**。本番では Let's Encrypt などを使います。

---

## 3-2. 自己署名証明書を生成する

証明書保存ディレクトリを作成：

```bash
sudo mkdir -p /etc/nginx/ssl
```

opensslで証明書と秘密鍵を一括生成：

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/server.key \
  -out /etc/nginx/ssl/server.crt \
  -subj "/C=JP/ST=Hokkaido/L=Sapporo/O=SchoolLab/CN=localhost"
```

| オプション | 意味 |
|------------|------|
| `-x509` | 自己署名証明書を生成 |
| `-nodes` | 秘密鍵にパスフレーズなし（Nginx起動時に入力不要） |
| `-days 365` | 有効期限365日 |
| `-newkey rsa:2048` | 2048ビットのRSA鍵を同時に生成 |
| `-subj` | 証明書の所有者情報（対話入力をスキップ） |

生成確認：

```bash
ls -la /etc/nginx/ssl/
```

`server.key` と `server.crt` の2ファイルが存在すればOK。

---

## 3-3. Nginx設定にHTTPS（443番ポート）を追加する

```bash
sudo nano /etc/nginx/conf.d/mysite.conf
```

以下のように全体を書き換える：

```nginx
# HTTP → HTTPS リダイレクト
server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

# HTTPS サーバー
server {
    listen 443 ssl;
    server_name localhost;

    # 証明書と秘密鍵の指定
    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    # TLSバージョンの制限（古いバージョンを無効化）
    ssl_protocols TLSv1.2 TLSv1.3;

    # リバースプロキシ（Tomcatへ転送）
    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 3-4. 設定を反映する

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 3-5. HTTPS接続を確認する

ブラウザで `https://localhost` にアクセスする。

> ⚠️ **「この接続は安全ではありません」警告が出る**
> → 自己署名証明書のため正常。
> → 「詳細設定」→「localhostにアクセスする（安全ではありません）」をクリックして続行。
> → TomcatのJSPページが表示されればOK。

---

## 3-6. curlで確認する（オプション）

```bash
# -k : 自己署名証明書の警告を無視
curl -k https://localhost

# HTTP→HTTPSリダイレクトを確認（-L はリダイレクト追跡）
curl -Lk http://localhost

# 証明書の詳細を表示
curl -vk https://localhost 2>&1 | grep -A5 "Server certificate"
```

---

## ゴール確認チェックリスト

| 確認項目 | 期待する結果 |
|----------|-------------|
| `http://localhost` | `https://localhost` へ301リダイレクトされる |
| `https://localhost` | 警告を経由してTomcatのJSPページが表示される |
| アドレスバー | 鍵マーク（または警告マーク）が表示される |
| `curl -kI https://localhost` | `HTTP/1.1 200 OK` が返る |
| `curl -I http://localhost` | `HTTP/1.1 301 Moved Permanently` が返る |

---

## まとめ

| 概念 | 内容 |
|------|------|
| `ssl_certificate` | 証明書ファイルのパスを指定 |
| `ssl_certificate_key` | 秘密鍵ファイルのパスを指定 |
| `ssl_protocols` | 許可するTLSバージョンを限定（セキュリティ強化） |
| `return 301` | HTTPアクセスをHTTPSへ永続リダイレクト |
| `X-Forwarded-Proto` | Nginxを経由していることをTomcatに伝えるヘッダー |

---

## 全体振り返り（3回の総括）

### 完成した構成

```
[ブラウザ]
    |
    | :80 → 301リダイレクト
    | :443 (TLS終端)
    |
[Nginx on WSL2]
    |  証明書検証・暗号化・復号
    |  proxy_pass（内部はHTTP）
    ↓
[Tomcat on Docker :8080]
    |
[hello.jsp]
```

### 役割分担の整理

| 役割 | 担当 |
|------|------|
| SSL/TLS終端（暗号化・復号） | Nginx |
| 静的ファイル配信 | Nginx |
| HTTP→HTTPSリダイレクト | Nginx |
| JSP実行・動的コンテンツ生成 | Tomcat |
| アプリの隠蔽（8080を外に出さない） | Nginxがプロキシすることで実現 |

### 本番環境との違い

| 項目 | 今回（学習） | 本番 |
|------|------------|------|
| 証明書 | 自己署名（openssl） | Let's Encrypt / 商用CA |
| ドメイン | localhost | 独自ドメイン |
| Nginxの場所 | WSL2ネイティブ | サーバーOS or コンテナ |
| Tomcatの8080 | ホストから直接届く | ファイアウォールで遮断 |

---

## 発展課題（任意）

1. `ssl_protocols` から `TLSv1.2` を外し `TLSv1.3` のみにして、古いクライアントとの互換性がどう変わるかを `curl --tls-max 1.2` で検証する。
2. `ssl_ciphers` ディレクティブを追加し、SSL Labs等の解説を参考に強い暗号スイートだけを許可する設定を試す。
3. 自己署名証明書をOSやブラウザの「信頼されたルート証明機関」にインポートして、警告が消えることを確認する（学習用端末のみで実施）。
4. Let's Encrypt（certbot）の運用フロー（ドメイン認証・自動更新）を調べ、自己署名証明書との違いをまとめる。

---

## トラブルシューティング

| 症状 | 確認ポイント |
|------|-------------|
| `https://localhost` が繋がらない | `sudo ss -tlnp \| grep 443` で443がLISTENしているか |
| `nginx: [emerg] cannot load certificate` | 証明書/鍵のパス・権限を確認（`ls -la /etc/nginx/ssl/`） |
| リダイレクトループ | 80側と443側の `server` ブロックを取り違えていないか |
| ブラウザが永遠にロード中 | Tomcatコンテナが落ちている可能性（`docker compose ps`） |

---

お疲れさまでした。これで **「ブラウザ → HTTPS → Nginx → リバースプロキシ → Docker上のTomcat → JSP」** という、実運用にも通じる基本構成を一通り自分の手で組めるようになりました。
