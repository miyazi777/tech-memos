

# opus 4.7

主要テーマ：4.6まで有効だった「6つの慣習」をやめるべき

- 細かい指示でのペアプロをやめる: 往復型プロンプトは性能低下の原因。初回に「目的・制約・完了条件」をまとめて提示し、委譲モデルへ転換する
- Effort Levelのmax常用をやめる: デフォルトのxhighが最適。maxは過度な思考傾向で精度が下がるリスクあり
- --dangerously-skip-permissionsの常用をやめる: Auto Mode(Max以上プラン)や/fewer-permission-promptsで安全に代替可能
- 長時間セッションの横付き監視をやめる: Focus Mode(/focus)で最終結果のみ表示、Recapsでセッション再開時にサマリー自動表示
- Subagentの毎回呼び出しをやめる: 並列作業や独立タスク以外は不要。Claude自身の判断に任せた方が高性能
- 検証機構なしの自律実行をやめる: テスト・スクリーンショット・期待出力を提供。Stop Hookでテスト自動実行が公式推奨で最も効果が高い



API仕様の重要変更

- fixed thinkingモード非サポート
- adaptive thinkingはデフォルトOFF
- temperature等の非デフォルト値はエラー
- トークン消費量が1.0〜1.35倍に変動

実装チェックリスト

- Claude Code v2.1.111以上に更新
- モデルをOpus 4.7、Effortをxhighに設定
- Stop Hookで検証ループを構築

その他の新機能

- /ultrareview：複数視点でのコード審査
- Task budgets：エージェント全体のトークン予算設定
- 高解像度画像対応：最大2576px／3.75MP

## /fewer-permission-promptsについて
セッションの履歴をスキャンして安全だが繰り返しパーミッションプロンプトを引き起こす一般的なbashおよぼMCPコマンドを見つける

## focus modeについて
/focus でセッションの途中作業を非表示にする。/focus でon offを切り替え可能

## recapsについて
/recap セッション再開時に過去のコンテキストを復元する


## 元記事
https://qiita.com/ot12/items/06420caf41a34a910c53
