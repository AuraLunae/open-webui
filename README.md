# open-webui
## Open WebUI + SearXNG ローカル構築 完全手順（Windows）

RTX 4050 Laptop 搭載機で、ローカルLLM＋Web検索付きチャット環境を構築するための一式です。

---

## 0. 構成の全体像

| 役割 | 使うもの | 実行場所 |
|---|---|---|
| チャットUI | Open WebUI | Dockerコンテナ |
| 検索エンジン | SearXNG | Dockerコンテナ |
| LLM実行基盤 | Ollama | ホストOS（Windows）に直接インストール |
| 生成モデル | Qwen3 8B | Ollama経由 |
| Embedding | Ruri base | Ollama経由 |
| リランカー | 日本語CrossEncoder | Open WebUI内部（CPU実行） |

処理の流れ：

```
質問
 ↓
Open WebUI が検索クエリを生成（日本語）
 ↓
SearXNG が検索 → タイトル / URL / スニペットを返す
 ↓
Open WebUI が上位URLに直接アクセスして本文を抽出（6件）
 ↓
リランカーが質問との関連度で並べ替え → 上位3件に絞る
 ↓
Qwen3 8B がその内容を読んで回答
```

---

## 1. 必要なソフトのインストール

### ① Docker Desktop
- https://www.docker.com/products/docker-desktop
- インストール時に **WSL 2** を選択（Hyper-Vではなく）
- インストール後にPCを再起動
- タスクバーのクジラアイコンが緑（Running）になっていることを確認

### ② Ollama
- https://ollama.com/download から Windows版をインストール
- デフォルト設定でOK。インストール後はバックグラウンドで常駐する

### ③ テキストエディタ（VSCode推奨）
- https://code.visualstudio.com/Download

---

## 2. モデルのダウンロード

PowerShell で実行します（初回は時間がかかります）。

```powershell
ollama pull qwen3:8b
ollama pull kun432/cl-nagoya-ruri-base
```

Ollamaが動いているか確認：

```powershell
curl http://localhost:11434/api/tags
```

JSONが返ってくればOK。

> **VRAMについての注意**
> RTX 4050 Laptop のVRAMは 6GB です。`qwen3:8b` はQ4量子化で約5GB前後あり、
> コンテキストを長く取ると一部がRAMにあふれて速度が落ちる場合があります。
> 動作が重いと感じたら `ollama pull qwen3:4b` に切り替えてください
> （4Bでも128Kコンテキスト対応で、十分実用的です）。

---

## 3. 作業フォルダの準備

```powershell
cd $env:USERPROFILE\Desktop
mkdir openwebui-setup
cd openwebui-setup
mkdir searxng
```

このフォルダに、同梱の `docker-compose.yml` と `.env` を配置します。

最終的な構成：

```
openwebui-setup\
├── docker-compose.yml
├── .env
└── searxng\          ← 最初は空。初回起動時に設定ファイルが自動生成される
```

---

## 4. 初回起動（設定ファイルの生成）

```powershell
docker compose up
```

SearXNGが `searxng\settings.yml` を自動生成します。
生成を確認したら `Ctrl + C` で一旦停止してください。

---

## 5. SearXNG の設定を修正【重要】

VSCodeで `searxng\settings.yml` を開き、2箇所修正します。

### ① JSON形式のレスポンスを有効化

`formats:` の行を探して、`- json` を追加：

```yaml
search:
  formats:
    - html
    - json
```

これがないと Open WebUI が検索結果を受け取れません。

### ② タイムアウトを延長

`outgoing:` セクションの `request_timeout` を10秒に：

```yaml
outgoing:
  request_timeout: 10.0
```

### ③（必要なら）レート制限を無効化

403エラーが出る場合は以下も変更：

```yaml
server:
  limiter: false
```

---

## 6. 本起動

```powershell
docker compose up -d
```

`-d` を付けるとバックグラウンドで起動します。

---

## 7. 動作確認

### SearXNG 単体の確認

ブラウザで http://localhost:8080 を開く。検索画面が出ればOK。

JSON応答の確認：

```powershell
curl "http://localhost:8080/search?lang=ja&q=テスト&format=json"
```

JSONが返ってくればOK。403やエラーなら手順5に戻ってください。

### Open WebUI の初期設定

1. ブラウザで http://localhost:3000 を開く
2. 「Get started」から管理者アカウントを作成
3. 左上のモデル選択で `qwen3:8b` を選ぶ

### Web検索の設定確認

「管理者パネル」→「設定」→「Web検索」で以下を確認：

- 「Web検索を有効にする」→ **オン**
- 検索エンジン → `searxng`
- SearXNG クエリURL → `http://searxng:8080/search?lang=ja&q=<query>`
- 保存を押す

> **なぜGUIでも設定するのか**
> Open WebUIの一部設定は「PersistentConfig」という仕組みで、
> **初回起動時に環境変数がDBへ保存され、以降は環境変数を変えても反映されません**。
> compose側を直しても効かない場合は、このGUIから直接設定するのが確実です。

---

## 8. 日本語で回答させる設定

「ワークスペース」→「モデル」→ `qwen3:8b` を選択し、
システムプロンプト欄に以下を入力して保存：

```
あなたは日本語で応答するアシスタントです。
ユーザーからの入力が何語であっても、必ず日本語で回答してください。
思考過程（thinking）も含めて、日本語で出力してください。
```

Qwen3は思考部分が英語になりがちなので、thinkingにも言及しておくのがポイントです。

---

## 9. 使い方

1. 新規チャットを開く
2. メッセージ入力欄の **「＋」ボタン** をクリック
3. メニューから「Web検索」をオン
4. 質問を送信

正しく動いていれば、回答の下に引用元リンクが表示されます。

---

## トラブルシューティング

### ケース1: `docker compose up` で "not found"

```powershell
docker --version
docker compose version
```

エラーが出る → Docker Desktopが起動していない。クジラアイコンを確認。

`no configuration file provided` → 実行フォルダが違う。

```powershell
cd $env:USERPROFILE\Desktop\openwebui-setup
dir
```

`docker-compose.yml` が表示されるか確認。

---

### ケース2: 「Web検索」トグルが見当たらない

- 常時表示のトグルではなく、入力欄の **「＋」メニューの中** にあります
- 管理者パネルで「Web検索を有効にする」がオフだと、そもそも項目自体が出ません

---

### ケース3: Web検索が動かない【今回発生した問題】

**症状**
検索をオンにしても何も起こらず、ログにも検索の形跡がない。

**原因**
環境変数名がバージョン間で変わっており、旧名称（`ENABLE_RAG_WEB_SEARCH`）
だけでは現行バージョンで検索処理が起動しませんでした。

**確認方法**

現在読み込まれている環境変数を確認：

```powershell
docker exec openwebui_host env | Select-String -Pattern "WEB_SEARCH|SEARXNG"
```

`ENABLE_WEB_SEARCH`（RAG_なし）が出てこなければこの問題です。

検索実行時のログだけを拾う：

```powershell
docker logs -f openwebui_host | Select-String -Pattern "search|searxng|ERROR|WARNING" -CaseSensitive:$false
```

**対処**
同梱の `docker-compose.yml` は新旧両方の環境変数を併記済みなので、
そのまま差し替えて再起動してください：

```powershell
docker compose down
docker compose up -d
```

それでも直らない場合は、手順7のGUI設定（PersistentConfig対策）を実施。

---

### ケース4: ログが大量で読めない

`GLOBAL_LOG_LEVEL` を `debug` にしているとSQLiteのポーリングログで埋まります。
同梱ファイルでは `info` にしてあります。デバッグが必要なときだけ `debug` に戻してください。

---

### ケース5: 検索結果の精度が低い

調整できるポイント：

| 設定 | 効果 |
|---|---|
| `WEB_SEARCH_RESULT_COUNT` | 取得候補数。増やすとリランカーの選択肢が広がる |
| `RAG_TOP_K` | LLMに渡す最終件数。増やすとコンテキストを圧迫 |
| `CHUNK_SIZE` | embeddingモデルのトークン上限に合わせる |
| Bypass Embedding and Retrieval | オンにすると本文全文をそのままLLMに渡す（長文注意） |

---

## 付録: 参考にした記事

- Qiita: Open WebUI で日本語ウェブ検索付き生成AIチャットを動かす（2025/3）
  https://qiita.com/RyoWakabayashi/items/95dea2b636039449f585
- Zenn: ローカル LLM と GUI で対話して、ウェブ検索もできるようにする（2025/8）
  https://zenn.dev/peaksandvalleys/articles/chat-with-local-llm-ollama-openwebui

いずれも執筆時点の環境変数名で書かれているため、そのままコピーすると
今回のケース3に該当する可能性があります。
