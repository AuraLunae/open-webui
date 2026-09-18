# searxng/settings.yml の修正箇所

初回 `docker compose up` すると、このフォルダに `settings.yml` が自動生成されます。
生成されたファイルに対して、以下の3箇所を修正してください。

**このファイル自体は参照用のメモです。settings.yml を上書きしないでください**
（自動生成された settings.yml には固有の `secret_key` が含まれているため）。

---

## ① JSON形式のレスポンスを有効化【必須】

`search:` セクション内の `formats:` を探します。
コメントアウトされているので、以下のように書き換えてください。

**変更前**

```yaml
search:
  # remove format to deny access, use lower case.
  # formats: [html, csv, json, rss]
  formats:
    - html
```

**変更後**

```yaml
search:
  # remove format to deny access, use lower case.
  # formats: [html, csv, json, rss]
  formats:
    - html
    - json
```

これがないと Open WebUI が検索結果を受け取れません。

---

## ② タイムアウトの延長【推奨】

`outgoing:` セクションの `request_timeout` を探します。
デフォルトは短いため、検索エンジンの応答が遅いとタイムアウトします。

**変更後**

```yaml
outgoing:
  # default timeout in seconds, can be override by engine
  request_timeout: 10.0
```

---

## ③ レート制限の無効化【403エラーが出る場合のみ】

`curl` でのJSONテスト時に 403 が返る場合は、`server:` セクションを変更します。

**変更後**

```yaml
server:
  limiter: false
```

ローカル環境のみで使う前提なので無効化して問題ありませんが、
外部公開する場合は有効のままにしてください。

---

## 修正後

```powershell
docker compose down
docker compose up -d
```

確認：

```powershell
curl "http://localhost:8080/search?lang=ja&q=テスト&format=json"
```

JSONが返ってくれば成功です。
