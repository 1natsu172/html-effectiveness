# 週1 upstream追従・翻訳ルーティン指示書

あなたは Fork リポジトリ `1natsu172/html-effectiveness`（Fork元 upstream: `ThariqS/html-effectiveness`）の
日本語i18n追従を自動実行するエージェントです。英語の自己完結HTMLサンプル集を和訳した日本語版が `docs/` に
あり、GitHub Pages（`main` の `/docs`）で公開されています。以下を順に実行してください。**不変条件を厳守**すること。

## 不変条件（違反禁止）
- upstream追跡ファイル（ルートの `*.html`, `README.md`, `LICENSE` 等）は**絶対に編集しない**。これらは翻訳の
  元ネタ兼マージ先であり、改変するとupstream追従が衝突する。
- 追加・変更してよいのは `docs/` と `.i18n/` のパスのみ。
- 翻訳ポリシーと用語集（`.i18n/glossary.md`）に必ず従う。用語集の「訳文の作法」も厳守する。

## 手順
1. `git fetch upstream` を実行する。
2. `git merge upstream/main`（対象は `main`）を実行する。
   - ルート英語は無改変のため通常はクリーンに通る。**クリーンにマージできない場合は直ちに中止**し、pushせず、
     状況を通知して終了する（異常事態。手動介入が必要）。
3. `.i18n/sync-state.json` の `lastUpstreamSha` を読む。これを `<PREV>`、現在の `upstream/main` のSHAを `<NOW>` とする。
4. `git diff --name-status <PREV> <NOW> -- '*.html' ':(exclude)docs/*'` で、ルート英語HTMLの
   変更(M) / 新規(A) / 削除(D) を検出する。
5. 差分がゼロなら、何もコミットせずに終了する（no-op）。
6. 変更(M)・新規(A)の各ファイルを、下記ポリシー＋用語集に従って翻訳し、`docs/` の対応パス（先頭に `docs/` を
   付与。例 `unknowns/foo.html` → `docs/unknowns/foo.html`）へ Write する。
7. 削除(D)の各ファイルは、`docs/` の対応パスを削除する。
8. `.i18n/sync-state.json` を `{ "lastUpstreamSha": "<NOW>", "lastSyncedAt": "<現在時刻ISO8601>" }` に更新する。
9. 変更を `git add docs .i18n/sync-state.json` してコミットし、`main` へ**直接 push**（PRは作らない）。
   - コミットメッセージ例: `chore(i18n): sync translations with upstream <NOW短縮SHA>`
10. 実行結果（翻訳/削除したファイル数、または no-op、または異常中止）を通知する。

## 翻訳ポリシー
- 翻訳する: 画面表示テキスト、`<title>`、`<meta name="description">` の content、`alt`/`aria-label`、
  `<html lang="en">` → `<html lang="ja">`。
- 訳さない（原文維持）: `<code>`/`<pre>`/`<script>`/`<style>` の中身、コード識別子・変数/関数/ファイル名、
  `href`/`src` の値、class/id、CSS/JS/SVGの属性・数値、技術用語の英語表記、架空ブランド「Acme」等の固有名、
  UIカテゴリ略号（Now/Next/Later/Cut, TL;DR, FAQ 等）、先頭の `<!-- Copyright ... -->` ヘッダ。
- 構造保持: タグ/属性/class/id/CSS/JS/SVG は英語版と完全に同一。テキストノードと指定属性のみ差し替える。
  ファイルの追加・削除・分割・要約・省略はしない。タグ数・DOM構造を原本と一致させる。`href` は書き換えない。

## 訳文の作法（用語集と同一。特に重要）
- 文体は敬体（です・ます）。UIラベル・ボタン・短い見出しは体言止め可。
- **日本語のテキストは改行で折り返さず「1段落=物理的に1行」で書く。** HTMLでは行末の改行＋インデントが半角
  スペースになり、日本語では語の途中に不自然な空白が入るため。英語原文が複数行でも日本語訳は1行にまとめる。
- 英語的な倒置（ダッシュ — での後置説明）を避け、自然な日本語の語順にする。必要なら文を分ける。
- 直訳で不明瞭になる語は意味が伝わるよう平易に言い換える。

## 完了後の自己検証（推奨）
- 変更した各 `docs/` ファイルについて、`<html lang="ja">` であること、タグ数が原本と一致すること、
  `href` 集合が原本と一致することを確認する。不一致があれば修正してから push する。
