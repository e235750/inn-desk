# 決済情報の自動収集・集計 計画

方針: 手入力しない。メール等から拾い、1つの台帳に寄せて月次・カテゴリで見る。

個人の取引明細は Git に置かない。正本は Google スプレッドシート。

## 目的

- 何にいくら使ったかを、後から追わない
- 注文・請求・返金を同じ物差しで見る
- 配送メールや広告で件数を膨らませない

## すでに分かっていること

メールを当たった結果、実際に動いている源は次のとおり。

| 源 | 使えるメール | 金額の取れやすさ | 注意 |
|---|---|---|---|
| Amazon | `auto-confirm@amazon.co.jp` の「注文済み」 | 本文の「合計」で取れる | 発送・配達メールは除外 |
| Steam | ご購入 / 返金完了 | スニペットに税込が出ることが多い。HTML 領収書はリンク先 | 購入と返金は別行。失敗カートは除外 |
| PayPayカード | `paypaycard-info@...` の「請求金額のお知らせ」（確定） | 月次合計は取れる | 内訳はカードサイト。仮確定メールは除外 |
| 三菱UFJカード | `notice@cr.mufg.jp` の請求額確定 | 金額がメールに無いことが多い | アプリ確認が必要。月次フラグだけ先に置く |
| Google Play | ご注文明細 | 定期購入は取れる | Gemini Plus は 2026-08-25 開始の試用（本請求は 2027-08-25 まで 0 円） |
| Microsoft | Game Pass 支払い・終了案内 | 支払いメールだけ計上 | 継続請求は無効、2026-09-18 終了予定 |
| PayPal | 支払いの領収書 | 取引単位で取れる | 月次口座明細はサマリなので個別領収書を優先 |
| GamersGate | 購入完了 | 注文 URL のみ。金額は本文に無い | 要フォロー or カード明細突合 |
| Patreon | クリエイター配信 | 決済メールではない | 対象外（課金通知が来たら追加） |
| Cursor Pro | 領収メールが見つかっていない | 未接続 | カード明細か請求ポータルが必要 |

カードの引き落とし口座は、PayPayカード分が沖縄銀行 読谷支店（普通）。カード番号は記録しない。

## 台帳（正本）

すでに空のシートを作った。

- 名前: `INN 決済台帳`
- ID: `1Bd5h_VsQSlfEVCjxPl9l9TVOLqwkG5AVmAmN5EtvJDs`
- URL: https://docs.google.com/spreadsheets/d/1Bd5h_VsQSlfEVCjxPl9l9TVOLqwkG5AVmAmN5EtvJDs/edit

シート構成（案）:

1. `transactions` … 1決済1行
2. `monthly` … 月 × カテゴリのピボット（数式）
3. `sources` … 収集クエリと最終実行日

### transactions 列

| 列 | 意味 |
|---|---|
| source_id | 重複排除キー（gmail thread id または注文番号） |
| booked_on | 発生日（JST） |
| merchant | Amazon / Steam / PayPayカード など |
| description | 商品名または請求月 |
| category | 下記の固定語 |
| amount_jpy | 支出は正、返金は負 |
| status | confirmed / pending / excluded |
| payment_method | カード名など（下4桁は持たない） |
| thread_id | Gmail 参照 |
| collected_at | 取り込み日時 |
| notes | 試用、突合待ちなど |

### カテゴリ（v1 は固定）

`subscription` / `game` / `amazon` / `travel` / `card_statement` / `other`

`card_statement` は月次請求の合計行。Amazon 個別行と二重計上しないため、集計ビューでは「明細ベース」と「カード引落ベース」を分ける。

## 重複と二重計上

これが一番壊れる点なので、ルールを先に固定する。

1. **同一注文のライフサイクル**  
   Amazon は「注文済み」だけを採用。発送・配達・レビュー依頼は捨てる。
2. **購入と返金**  
   別 `source_id` の2行。ネットは合計で見る。
3. **カード請求 vs 店の領収**  
   店の領収（Amazon / Steam / PayPal）を明細の正とする。  
   PayPay・MUFG の月次メールは「引落確認」用で、月次合計の検算にだけ使う。  
   同じ支出を両方 `confirmed` にして月次合計に足さない。
4. **失敗・保留**  
   Steam の購入エラー、返金リクエスト受付は `excluded` または無視。
5. **試用**  
   金額 0 で `pending`。本請求が来たら更新。

## 自動収集の仕組み（段階）

完全無人の常時クロールは、今の inn-desk には常駐ジョブが無い。v1 は「エージェントが決まった手順で回す」にする。

### v1 — エージェント収集（今回の計画の本体）

1. 決まった Gmail クエリでスレッドを取る
2. 件名・差出人でパーサを選ぶ
3. `source_id` がシートに無ければ追記
4. `monthly` を再計算
5. 金額が取れなかった行は `pending` にして残す

推奨クエリ:

```
from:auto-confirm@amazon.co.jp subject:注文済み newer_than:90d
from:noreply@steampowered.com (subject:ご購入 OR subject:返金が行われました) newer_than:90d
from:paypaycard-info@mail.paypay-card.co.jp subject:請求金額 newer_than:1y
from:googleplay-noreply@google.com subject:ご注文明細 newer_than:1y
from:service-jp@paypal.com subject:領収書 newer_than:1y
from:support@gamersgate.com subject:purchased newer_than:1y
from:microsoft-noreply@microsoft.com (subject:支払い OR subject:ご注文) newer_than:1y
```

inn-desk に置くもの:

- パーサ（Amazon / Steam / PayPay月次 / Google Play / PayPal）
- 重複判定とカテゴリ推定
- エージェント用スキル（手順・クエリ・シート ID）
- 匿名化した fixture テスト
- 取引実データは置かない

### v2 — 常時自動

自宅 Docker 上で定期実行し、同じパーサでシートに追記する。  
カード内訳が必要なら、スクレイピングより明細書 CSV の手動ドロップを先に検討する。

### v3 — カード明細の突合

PayPay / MUFG の CSV を取り込み、メール明細と `booked_on` + 金額で突合。  
GamersGate や Cursor の穴をここで埋める。

## やらないこと（v1）

- 取引明細をリポジトリや INN 本文に保存する
- 配送・広告・Patreon 投稿を決済として数える
- カード番号・口座番号の全文を記録する
- 銀行 API やカードサイトへのログイン自動化
- 税務申告用の証憑保管（必要なら後で Drive フォルダを分ける）

## 成功条件

- 直近 90 日の Amazon「注文済み」と Steam 購入/返金が、手入力なしで台帳に載る
- 同じ注文を2回足しても行が増えない
- 月次ビューで「店明細合計」と「PayPayカード請求」を並べて差が見える
- 金額不明は消えず `pending` のまま残る

## 確認したいこと（実装前）

計画としては上で進める。次だけ後から変えてよい。

1. カテゴリを細かくするか（バイク / サプリ など）。v1 は粗くてよい想定
2. カード月次を検算専用にするか、家計の「支出」そのものにするか。計画では検算専用
3. v1 をエージェント手動実行のままにするか、すぐ Docker cron までやるか。計画ではエージェント先行

## 実装順（計画承認後）

1. パーサとテスト
2. スキル（収集手順）
3. シートの見出しと `monthly` 数式
4. 直近 90 日を1回流して検算
5. 金額不明（GamersGate、Cursor、MUFG）を pending 一覧にする
