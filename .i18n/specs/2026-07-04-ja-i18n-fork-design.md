# ja_JP i18n Fork + 週1自動追従 設計spec

- 日付: 2026-07-04
- ステータス: 承認済み（実装プラン作成へ）
- 対象リポジトリ: `1natsu172/html-effectiveness`（`ThariqS/html-effectiveness` のFork）

## 背景・目的

本リポジトリは「HTMLをClaudeの出力フォーマットとして使う」ブログ記事に付随する、
自己完結型HTMLサンプル集（全33ファイル）のForkである。内容は英語話者前提の英語で
書かれており、日本語話者が直接読むには不便。そこで **英語の内容を和訳した日本語版HTML**
を用意し、GitHub Pagesで日本語サイトとして公開する。あわせて、Fork元（upstream）の更新に
**週1でクラウド上から自動追従し、差分を再翻訳して自動コミット**する運用を確立する。

upstreamは README に "Not maintained and not accepting contributions" と明記されており、
更新頻度は低い見込み。週1追従は「更新があれば拾う保険」の位置づけ。

## ゴール / 非ゴール

### ゴール
- 33ファイルすべての日本語版を用意し、GitHub Pagesで**開いたら即日本語**の状態にする
- upstream更新への追従を**コンフリクトゼロ**で回せる構造にする
- 週1のクラウド定期実行で「追従→差分翻訳→自動commit&push」を**完全ハンズオフ**で実現する

### 非ゴール
- EN/JA言語切替ナビゲーションのHTMLへの作り込み（独自UIは持ち込まない）
- 英語版のGitHub Pages公開（英語はリポジトリ内の「翻訳の元ネタ」としてのみ保持）
- upstreamへのコントリビュート（upstreamは受け付けていない）

## アーキテクチャ決定

### 中核制約：gitマージはパス単位

gitのマージはパス単位で対応を取る。upstreamの `index.html` はこちらの同名パス
`index.html` にマージされる。したがって **ルートのファイルを日本語で上書きすると、
upstreamが同ファイルを更新するたびに必ず衝突する**。これはマージの仕組み上の必然であり、
運用では回避できない。

コンフリクトゼロの必要条件は「**upstream由来のファイル（ルートの英語HTML）を一切改変
しないこと**」。よって日本語は別パスに「複製」として持つ。

### 採用案：`docs/` を日本語サイトルートとする

GitHub Pagesの「Deploy from a branch」で選べる公開フォルダは仕様上 `/（ルート）` か
`/docs` の2択のみ。任意フォルダ（`/ja` 等）は選べない。そこで **日本語訳を `docs/` に
配置し、Pagesの `/docs` ネイティブ配信を使う**。これにより GitHub Actions もブランチ
切替も不要（＝独自の仕組みを最小化）で、単一ブランチ `main` に英語原本と日本語訳が
並び、自動追従の監査性も高い。

- `docs/` という名前が意味的にやや変（本来「ドキュメント」）だが、割り切って運用する
- upstreamが将来 `docs/` を追加した場合のみ衝突しうるが、unmaintained明記のため実質ゼロリスク

### 検討したが採らなかった案
- **ルート上書き**: 開いて即日本語のUXは最良だが、upstream更新のたびに全触ファイルが衝突。却下
- **`ja/` サブディレクトリ + GitHub Actionsデプロイ**: 監査性は同等だがデプロイにActionsが要り煩雑。`docs/`案で同等の利点が得られるため不要
- **`ja`ブランチをPages配信**: デプロイは素だが、追従時に `git merge main` すると同一パス衝突が再発する footgun があり、履歴も2本に割れる。却下

## リポジトリ構成

```
リポジトリ（main 単一ブランチ）
├ index.html          英語・upstream原本（無改変＝翻訳元・マージ先）
├ 01-....html 〜 20-....html   英語（無改変）
├ unknowns/*.html     英語（無改変）
├ README.md 等        upstream追跡ファイル（無改変）
├ docs/               日本語訳（Pagesのルートとして配信）
│  ├ index.html
│  ├ 01-....html 〜 20-....html
│  ├ unknowns/*.html
│  └ .nojekyll        Jekyll処理を無効化（静的HTMLの安全策）
└ .i18n/              Fork独自メタ（新規パス・Pages非公開）
   ├ glossary.md          用語集（訳語統一）
   ├ sync-state.json      最終同期upstream SHA
   ├ routine-prompt.md    週1ルーティンが実行する指示書
   └ specs/               設計spec（本ファイル）
```

### Fork衛生ルール（不変条件）
- **upstream追跡ファイル（ルートの `*.html`, `README.md` 等）は絶対に編集しない。**
  編集＝コンフリクトの原因。Fork独自の追加物はすべて**新規パス**（`docs/`, `.i18n/`）に置く。
- 相対リンク（`href="01-...html"`, `href="unknowns/index.html"` 等）は `docs/` に構造ごと
  複製すれば内部で完結する。絶対パスリンク（`/...`）は本リポジトリに存在しないことを確認済み。

## 翻訳ポリシー

### 文体
- **敬体（です・ます調）** を基本とする
- UIラベル・ボタン・見出し等の短い語句は体言止めで自然にする

### 翻訳対象
- 画面表示テキスト（本文・見出し・キャプション等）
- `<title>`、`meta[name=description]` 等の表示・要約テキスト
- `alt` / `aria-label` 等のアクセシビリティテキスト
- ルート要素の言語属性 `<html lang="en">` → `<html lang="ja">`

### 原文維持（翻訳しない）
- コードブロック、コード識別子、変数名・関数名・ファイル名
- 技術用語の一般的な英語表記（文脈で自然な場合）
- 架空ブランド名「Acme」等の固有名、サンプルデータ中の固有名詞
- 先頭の Anthropic 著作権ヘッダ（`<!-- Copyright ... -->`）
- CSS/JavaScript のロジック（表示文字列の翻訳は除く）

### 用語集による訳語統一
- `.i18n/glossary.md` に訳語対応表を持ち、**初回一括翻訳と週1ルーティンの両方が参照**する。
  33ファイル間および将来の差分翻訳で訳語がブレないようにするための単一の情報源。
- 初期エントリ例（実装時に精査・拡充）:
  - pull request → プルリクエスト（PR）
  - commit → コミット
  - diff → 差分
  - review → レビュー
  - deploy / deployment → デプロイ
  - incident → インシデント
  - rollback → ロールバック
  - feature flag → フィーチャーフラグ
  - milestone → マイルストーン

## 初回の一括翻訳（本セッションで実施）

- 33ファイルは互いに独立した自己完結HTMLのため、**並列サブエージェントでfan-out**して
  `docs/` を一気に生成する（`Workflow` ツールを使用）。
- 各エージェントは「1ファイル（または少数束）」を担当し、翻訳ポリシー＋用語集に従って
  英語ファイルを翻訳し、`docs/` の対応パスへ出力する。
- `Workflow` は多エージェント＝トークン多消費のため、**実行直前にユーザーの明示的なGO**を
  得てから起動する（無断では起動しない）。
- 完了後、`docs/.nojekyll` を配置し、`.i18n/sync-state.json` に現在のupstream SHAを記録する。

## 週1ルーティン（Claudeクラウド定期実行）

### スケジュール
- 毎週 **月曜 09:00 JST**

### 処理フロー
1. `upstream/main` を fetch し、`main` へマージする。ルート英語は無改変のため**クリーンに通る想定**。
   万一クリーンでない場合は**強行せず処理を停止し、ユーザーに通知**する（異常事態）。
2. `.i18n/sync-state.json` 記録のSHA 〜 最新upstream SHA で、ルート英語ファイルの
   **変更(M) / 新規(A) / 削除(D)** を検出する。
3. 変更・新規 → 翻訳ポリシー＋用語集に従い `docs/` の対応パスへ翻訳出力。
   削除 → `docs/` の対応ファイルも削除。
4. `.i18n/sync-state.json` を最新SHAへ更新。
5. `main` へ**直接 commit & push**（PRなし・完全ハンズオフ）。Pagesが自動再ビルド。
6. **差分ゼロなら no-op**（コミットしない）。**失敗・異常時のみ通知**。

### 状態管理
- `.i18n/sync-state.json` に最終同期情報を保持・コミットする。形式:
  ```json
  {
    "lastUpstreamSha": "<40-hex sha>",
    "lastSyncedAt": "<ISO 8601>"
  }
  ```
  次回実行はこの `lastUpstreamSha` を起点に差分を検出する。

### 指示書
- ルーティンが毎回実行する具体手順は `.i18n/routine-prompt.md` に記述し、routineから参照する。
  翻訳ポリシー・用語集・Fork衛生ルール・停止条件（マージ非クリーン時）を明記する。

### 使用しないもの
- GitHub Actions は使わない（Pagesは `/docs` ネイティブ配信、追従はClaudeクラウドルーティン）。

## GitHub Pages 設定

- Settings → Pages → **Source: Deploy from a branch**、Branch: `main`、Folder: `/docs`
- 公開URL（例）: `https://1natsu172.github.io/html-effectiveness/` → `docs/index.html`（日本語）

## セットアップ前提・確認事項

- **クラウド書き込み権限**: 週1ルーティンは `origin` へ push するため、Claudeクラウド実行環境に
  本リポジトリへの **write 権限** が必要。routine登録時に確認する。
- GitHub Pages の有効化（上記Source設定）は初回セットアップで実施する。

## リスクと対処

| リスク | 対処 |
|---|---|
| upstreamが将来 `docs/` を追加し衝突 | unmaintainedのため実質ゼロ。発生時に個別対処 |
| マージが非クリーン（想定外の改変・履歴書換） | ルーティンは強行せず停止＋通知。手動介入 |
| 自動翻訳の品質ブレ | 用語集で訳語統一。初回は人間レビュー可能。異常時通知 |
| Fork衛生ルール違反（ルート英語を誤編集） | ルーティン指示書に不変条件として明記。編集は`docs/`・`.i18n/`限定 |

## 実装の分割（後続のwriting-plansで詳細化）

1. **基盤セットアップ**: `docs/.nojekyll`、`.i18n/glossary.md`、`.i18n/sync-state.json`、
   `.i18n/routine-prompt.md` の雛形作成。GitHub Pages設定手順の用意。
2. **初回一括翻訳**: `Workflow` で33ファイルを並列翻訳し `docs/` を生成（ユーザーGO後）。
3. **週1ルーティン登録**: routine指示書を確定し、月曜09:00 JSTのクラウド定期実行を登録。
