---
title: "RenderのCron JobsでLINE Botから定期的にメッセージを配信する"
emoji: "🕒"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linebot", "render", "cron"]
published: true
---

[Render](https://render.com/)の**Cron Jobs**を使って、LINE Botから定期的にメッセージを送る設定をしてみました。
Renderアカウントの準備、Githubからのデプロイ、Cron Jobsの定期実行までを本記事で扱います。
LINE Botに限らず、Renderでスケジュール実行を行いたい場合に参考にしていただければと思います。

なお、LINE BotはGoで実装しましたが、Botの実装自体は本記事では扱いません。

## Renderの概要
Renderは様々なWebアプリをデプロイできるPaaSで、Herokuの代替となるようなサービスです。
![](/images/run-line-bot-with-render-cron/render-image.png)
Webサーバーのデプロイや静的サイトのホスティング、バックグラウンドジョブ、Cron Jobs、自動バックアップ機能付きDBなどが揃っています。
Githubから簡単にデプロイできます。

https://render.com/

なお、Herokuとの比較については、公式の[こちらのページ](https://render.com/render-vs-heroku-comparison)にも書かれています。

## Renderアカウントの準備
[こちら](https://dashboard.render.com/register)からRenderアカウントを作成します。
Github経由でアカウントを作成すると楽です。
Github認証・メール認証をするとダッシュボードにアクセスできます。

## Cron Jobsを作成
以下公式ドキュメントです。
https://render.com/docs/cronjobs

#### 対象
今回は以下のレポジトリを使用します。
https://github.com/t-shimpo/spotify-new-release-bot

#### 手順

[ダッシュボード](https://dashboard.render.com/)から`New Cron Job`ボタンをクリックします。
![](/images/run-line-bot-with-render-cron/cron-jobs1.png)

**Cron Jobs**は無料のサービスではないため、クレジットカードの登録が求められます。
![](/images/run-line-bot-with-render-cron/cron-jobs2.png)

クレジットカードを登録すると、Cron Job作成画面に遷移します。
![](/images/run-line-bot-with-render-cron/cron-jobs3.png)

レポジトリと接続するか、パブリックなレポジトリのURLから設定するかの2種類の方法が用意されていますが、今回はレポジトリと接続する方法にしました。
画面右の`Connect account`をクリックして設定できます。
設定すると、以下のようにレポジトリが表示されますので、`Connect`ボタンをクリックします。
![](/images/run-line-bot-with-render-cron/cron-jobs4.png)

すると設定画面に遷移しますので、`Name`、`Build Command`、`Schedule`、`Command`などの必要項目を入力します。

![](/images/run-line-bot-with-render-cron/cron-jobs5.png)

また、今回は環境変数を使用しているため、`Advanced`をクリックし、`Add Environment Variables`から環境変数の登録も行いました。

![](/images/run-line-bot-with-render-cron/cron-jobs6.png)

入力を終えたら、`Create Cron Job`をクリックします。

以下の画面に遷移し、cronの状態を確認できます。
Buildも自動で行われます。
![](/images/run-line-bot-with-render-cron/cron-jobs7.png)


**LINE画面**
実際にメッセージが送信されることも確認できました。
![](/images/run-line-bot-with-render-cron/line-screen.png)


## 参考資料

https://render.com/
https://render.com/docs/cronjobs
