# Handwritten Scorebook Reader — 手書き野球スコアブック読解 R&D（技術解説）

紙の草野球スコアブックの写真から、打席・走塁データを構造化抽出する研究開発プロジェクトの公開技術解説です。

**本体リポジトリ: [`baseball-scorebook-ocr`](https://github.com/yasumorishima/baseball-scorebook-ocr) 🔒 private** — 記法解読・ソルバー設計・ground truth 転記規約などのノウハウと実データは非公開です（the method is the product）。この repo では公開できる範囲の設計思想・実績・開発プロセスを紹介します。

> **English summary**: Reads handwritten Japanese amateur-baseball scorebooks from photos into structured at-bat data — no paid APIs, no cloud OCR. Deterministic computer vision (OpenCV on a Raspberry Pi 5) fused with a base-running constraint solver that only accepts readings consistent with legal baseball plays. 82% pooled occupancy accuracy on a 25-game hand-transcribed ground-truth corpus (the complete archive); 23 consecutive held-out sheets without a single falsification of the frozen constraint model. The main repo is private — this is the public write-up.

## なぜ「未解決領域」なのか

日本の野球スコアブックは独自の記号体系（守備番号の連鎖、塁到達マーク、アウトを表すローマ数字、盗塁・暴投などの機構表記…）を持ち、そこに記録者ごとの手書きの揺れ・省略・書き損じ・シート品質差が重なります。汎用 OCR やマルチモーダル LLM に写真を丸ごと渡しても実用精度は出ません（実測して確認済み）。「どの AI もまだ解けていない」ドメインです。

## アプローチ: 有料 LLM Vision から、無課金 CV + 制約ソルバーへ

初期は LLM Vision に前処理済みクロップを渡す方式を検証しましたが、2026-07 に方針を全面転換し、**API 課金ゼロの決定論的パイプライン**を構築しました。

```
写真 → 格子検出（deskew 込み）→ 手書きインク分離 → テンプレートマッチング（kNN）→ 塁占有制約ソルバー
```

鍵は最後の**制約ソルバー**です。テンプレート認識単体では最難クラスのマークは 4 割程度しか当たりませんが、「野球のルール上、合法な走塁として成立する読みしか受理しない」という制約層を重ねることで、複数の候補読みの中から唯一の整合解を選び出します。曖昧な手書きグリフも、イニングのアウト数・得点・走者の追い越し禁止といった試合の算術が裁定してくれます。

## 実績（2026-07 時点）

- ground truth: **25試合分を全打席レベルで手転記（原本アーカイブ完結）**（複数の記録者・筆記具・シート品質。促進タイブレークや両面ペアなどの特殊ケースを含む）
- 塁占有の総合精度: **pooled 82%**（テンプレ認識単体 54% → ソルバー統合で +29pt）
- 認識層の改良3段（2026-07）: 記法慣習に根ざした距離項の追加、注記の混入を防ぐ**切り出しの多候補化**、字形に重なった別マークを剥がす外科的切り出しで認識段を 41%→54% に改善。最終段は 80%→82%（制約ソルバーの組み替えノイズ1振幅ぶんの上振れ＝「有望・未証明」として正直に記録し、頑健な成果は認識段側とする。3段目は入れ替わりゼロのクリーンな +1 セル）
- **モデル凍結後に転記した held-out 23枚連続で、制約モデルが一度も反証されていない**（パーフェクト読解のシートも出現）

## 独立データとの相互検証（2026-07）

チームサイトに手入力されていた試合別成績（別の人手・別の情報源）と、GT 由来の選手別成績を9試合ぶん突合しました。**7/9 試合でスコアが完全一致、選手単位でも大部分のスロットが桁単位で一致**（単打+三塁打+本塁打+四球+打点5 という行まで一致）。さらに GT 側で「判読曖昧」とフラグしていたグリフ5つが、シート自身の集計行とサイトデータの両方から同じ向きに確定しました。逆にサイト側の入力ミス候補（スコアの1点違い・交代選手の打席二重計上・死球の記録漏れなど）も特定でき、**相互検証が双方向に機能する**ことを確認しています。残っていた食い違い2試合も原本の再精査で決着し（うち1試合は転記時に見落としていたシート自身のラインスコアが裁定者になり、GT 側を訂正）、紙で答えられる疑問は全てクローズしました。

## 開発プロセスの規律

- **frozen model + held-out 評価**: 認識モデルは凍結し、新しく転記するシートはまず「答えを見せない汎化テスト」として評価してからテンプレートに取り込む
- **honesty ledger**: テンプレート追加のたびに「直った旧ミス」と「新たに壊れたセル」を両方記録する（テンプレ増は非単調に効く。良い話だけを書かない）
- **敵対的レビュー**: 転記データは毎回 LLM による整合レビューにかけ、走塁の合法性・得点の算術を突合する（実際に人間の転記バグを検出した実績あり）。ただし認識パイプライン自体に生成 AI は使わない
- **個人情報の分離**: 選手名は転記データに含めず、スコアブック画像・実試合データはリポジトリ外で管理する

## 応用先

草野球チーム [横浜ファニーズ公式サイト](https://github.com/yasumorishima/yokohama-funnies-docs)の試合別成績データ基盤。2026年シーズンの打撃集計は本パイプラインの ground truth データから作成済みです。

## 環境

Raspberry Pi 5 上の Python + OpenCV（headless）。クラウド・有料 API は不使用です。

---

ソースコードと手法の詳細（記法解読モデル・ソルバーの制約設計・転記規約）は [private リポジトリ](https://github.com/yasumorishima/baseball-scorebook-ocr) 🔒 にあります。興味のある方は [GitHub profile](https://github.com/yasumorishima) からどうぞ。
