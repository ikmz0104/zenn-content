---
title: "AdoのレディメイドとJ.Y.ParkのSwing Babyのイントロが似ていたので検証してみた"
emoji: "🎧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Spotify", "Python"]
published: false
---

# 背景
SpotifyでAdoさんの曲をシャッフルして聴いていたら、なんだか聞き覚えのある曲だったので調べると、NiziProjectでチームお姉さんが披露したJ.Y.ParkのSwing Babyという曲だった。Spotify-APIを使えばどれくらい似ているかが分かるのかなあと思い手を動かした。

# 環境
- Python (Google Colab)

# プロジェクト設定からデモテスト
2つの曲をプレイリストにまとめてSpotify-APIを使用すると２曲の全体曲調を比較することは容易だった。
https://zenn.dev/mizuneko4345/scraps/7c4fda63358a01

問題は２曲のイントロ部分にターゲットを絞り込んで音源分析を行えるかということ。

# ターゲットを絞り込む方法
