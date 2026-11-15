# Handwritten Scorebook Reader — 手書き野球スコアブック読解 R&D（技術解説）

紙の草野球スコアブックの写真から、打席・走塁データを構造化抽出する研究開発プロジェクトの公開技術解説です。

**コード: [`baseball-scorebook-reader`](https://github.com/yasumorishima/baseball-scorebook-reader)（公開・MIT）** — 生成 AI を使わずに読む部品（`cv/`）と、Claude で読むツール（`scorebook/`）。選手名を含む正解データとスコアブックの画像は非公開の [`baseball-scorebook-ocr`](https://github.com/yasumorishima/baseball-scorebook-ocr) 🔒 に置いています。この repo では設計の考え方・実績・開発の進め方と、スコアブックを読む・まとめる人向けの資料を公開しています。

- **[手書きのスコアブックの記号の読み方](notation.md)**：1 マスの見方、打者の結果・走者の進みの書き方、実物で出会った記録者ごとの書き方の違い
- **[スコアブックから個人成績をまとめる手順](stats-from-scorebook.md)**：読んで登録する流れと、数字が合っているかの検算
- **[Claude Code のスキルのひな形](claude-code/scorebook-stats/SKILL.md)**：写真を貼るだけで、読み取り・検算・登録まで進めるための手順書

> **English summary**: Reads handwritten Japanese amateur-baseball scorebooks from photos into a line score and per-player batting lines. Since 2026-10 the reader is Claude (Opus) on column crops cut out by our own grid detection, with a second Opus read that flags cells where the two reads disagree. On 33 hand-transcribed sheets Claude alone gets 370/396 corner digits (93%) against 283 for our own no-hints computer vision; plate-appearance results 859/941 and the line score 25/33 sheets. Against the team site's hand-entered stats (19 games, 185 batting-order rows) all main counts match on 153 rows (83%) and the line score on 17/19 games; checking every difference on paper also found about 20 input errors on the site. About $0.6 per game. Every API run is pre-registered with a cost cap. The earlier no-API pipeline (OpenCV on a Raspberry Pi 5 plus a base-running constraint solver) is kept below as the development record. Photos and ground truth with real names stay in a private repo.

## なぜ「未解決領域」なのか

日本の野球スコアブックは独自の記号体系（守備番号の連鎖、塁到達マーク、アウトを表すローマ数字、盗塁・暴投などの機構表記…）を持ち、そこに記録者ごとの手書きの揺れ・省略・書き損じ・シート品質差が重なります。汎用 OCR やマルチモーダル LLM に写真を丸ごと渡しても実用精度は出ません（実測して確認済み）。「どの AI もまだ解けていない」ドメインです。

## アプローチ

**2026-10 から、読み手は Claude（Opus）です**（下の「実績（2026-11 時点）」）。写真から格子を自前で検出し、列ごとに切り出した画像を Claude に読ませ、打席・得点・角の数字から得点表と打順ごとの成績を組み立てます。同じ列をもう一度 Opus に読ませ、2 回の読みが割れたマスに印を付けます。

それまで（2026-07〜10）の、API を使わない方式（OpenCV＋野球の決まりで解く制約ソルバー・Raspberry Pi 5）の説明と開発記録は [history-cv-solver.md](history-cv-solver.md) にまとめました。

## 実績（2026-11 時点）

数字はすべて Claude Code cloud で測ったもの。正解データ（33 枚）は人の手で全打席を書き起こしたもの、サイトの成績は別の人がチームのサイトに手入力したものです。

- **角の数字（塁を進んだ打席の番号など・396 か所）**：Claude が何も渡されずに **370（93%）**。自作 CV（何も渡さない）は 283（71%）、自作の論理（ソルバー）を足しても Claude の読みはほとんど良くならない（2026-10-14・事前登録）。
- **得点・三振も読ませた読み方（2026-10-21）**：打席の結果 **859/941**、三振 132/133、得点表が正解と全部一致 **25/33 枚**（読み方を変える前は 16/33）。
- **チームのサイトとの照合（2026-11-08・19 試合）**：打順の行 185 のうち、主な 8 項目（打席・打数・安打・二塁打・三塁打・本塁打・四死球・三振）が全部一致 **153 行（83%）**、回ごとの得点が一致 **17/19 試合**。両チームの用紙のどちらが自チームかは、サイトの回ごとの得点で自動で選べた（19/19）。
- **違いは全部、用紙で確かめる**：違いの多くはツールの外れでしたが、サイト側の入力ミスも約 20 件見つかり（盗塁・三振の記入漏れ、抜けた打席、打点など）、チームの担当者に伝えました。
- **規則に合わせた直し（2026-11-08）**：タイブレークで置かれた走者の得点を数える。打点は公認野球規則 9.04 どおり（失策・暴投・盗塁で還った点は原則打点にしない）。決めた材料と別の試合で確かめてから入れました。
- **費用**：1 試合（両チームの用紙＋自チームの 2 回目の読み）で約 $0.6。API を呼ぶ前に毎回、上限の費用を書いた事前登録を置き、回数と費用を記録しています（19 試合の全数テストは 1311 回・$12.19）。
- **試したが採らなかったもの（2026-10-28）**：読み手を別のモデルに替える・2 回目の読みを軽いモデルにする・3 回読んで多数決、など。どれも事前登録した条件に届かなかった。

## 開発プロセスの規律

- **敵対的レビュー**: 転記データは毎回 LLM による整合レビューにかけ、走塁の合法性・得点の算術を突合する（実際に人間の転記バグを検出した実績あり）。2026-10 からは読み手そのものも Claude（Opus）で、2 回目の読み（Opus による見直し）で食い違うマスに印を付ける
- **API は事前登録してから**: 呼ぶ前に、何を測るか・成立の条件・費用の上限を書いて凍結し、結果を見てから条件や正解を変えない
- **個人情報の分離**: 選手名は転記データに含めず、スコアブック画像・実試合データはリポジトリ外で管理する

## 応用先

草野球チームのサイトの試合別成績データ基盤。2026年シーズンの打撃集計はこれまで手で書き起こした正解データから作成済みです。2026-11 からは、新しい試合の用紙を読んでサイトの成績と照らし合わせ、サイトの入力ミスを見つける用途にも使っています。

## 環境

Python + OpenCV（headless）。格子の検出と切り出しは自前、読み取りは Claude API（Opus）。開発は Raspberry Pi 5 と Claude Code cloud の両方（2026-10 から）。API を使わない CV＋ソルバーの部品も残しています。

---

コードは [baseball-scorebook-reader](https://github.com/yasumorishima/baseball-scorebook-reader)（MIT）、記号の読み方は [notation.md](notation.md)、成績をまとめる手順は [stats-from-scorebook.md](stats-from-scorebook.md) にあります。API を使わない方式の開発記録は [history-cv-solver.md](history-cv-solver.md) にあります。選手名を含む正解データと画像だけは非公開です。
