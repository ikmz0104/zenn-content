---
title: "【Python】ハッシュツリー 実装しました"
emoji: "🌲"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Blockchain", "Bitcoin", "Python"]
published: false
---

# はじめに
ブロックチェーン技術に関する記事を読んでいると頻繁にハッシュツリーという単語がでてくるので理解のため実装して動作確認してみました。
こちらが今回実装＆動作確認したソースです。
https://github.com/ikmz0104/merkle-root/blob/main/main.ipynb
# 環境
Jupyter Notebook

# 言葉の整理
## ハッシュツリー
ハッシュリストをペアにしてハッシュ化していき求まった一つのハッシュと元データの構造の整合性を見ることで、改ざんが行われていないか検証する技術のこと。

この際に必要なのが、順序的ハッシュリスト（ブロック内のトランザクション）と暗号学的ハッシュ関数（hash256 等）です。

処理フローは下記の通りです。
1. 好きなハッシュ関数を使って、順序的ハッシュリストの要素全てをハッシュ化する
2. ハッシュの個数が奇数なら末尾のハッシュを複製して偶数にする
3. ハッシュをペアにして連結し新たなハッシュを生成
4. 2～4を繰り返す（ハッシュの個数が1になれば終了）

## マークルペアレント
「ハッシュリストをペアにしてハッシュ化していく」と言いましたが、２つのペアとなるハッシュについてはそれぞれ左ハッシュ(L)・右ハッシュ(R)と呼び、左右のハッシュを連結した(H)結果できるのものを親ハッシュ(P)と呼びます。実装としてはhashが一つになるまでwhileループを回すようなイメージです。
```
P = H(L||R)　※||は連結
```
## リーフ
最下層のハッシュリストのこと（後で触れます）

## 内部ノード
リーフとマークルルート以外のハッシュリストのこと（後で触れます）

## マークルペアレントレベル
各ペアの親ハッシュを得るための条件のこと。それは、順序付きハッシュリストが３つ以上存在することです。

## マークルルート
最高位の親ハッシュのこと。こいつを求めるまで順序付きハッシュリストをペアにしてハッシュ化していきます。

## 包含証明
https://zenn.dev/link/comments/4b76af95d48dd4

# 動作確認

使用する順序付きハッシュリスト
```
hex_hashes = [
    "9745f7173ef14ee4155722d1cbf13304339fd00d900b759c6f9d58579b5765fb",
    "5573c8ede34936c29cdfdfe743f7f5fdfbd4f54ba0705259e62f39917065cb9b",
    "82a02ecbb6623b4274dfcab82b336dc017a27136e08521091e443e62582e8f05",
    "507ccae5ed9b340363a0e6d765af148be9cb1c8766ccc922f83e4ae681658308",
    "a7a4aec28e7162e1e9ef33dfa30f0bc0526e6cf4b11a576f6c5de58593898330",
    "bb6267664bd833fd9fc82582853ab144fece26b7a8a5bf328f8a059445b59add",
    "ea6d7ac1ee77fbacee58fc717b990c4fcccf1b19af43103c090f601677fd8836",
    "457743861de496c429912558a106b810b0507975a49773228aa788df40730d41",
    "7688029288efc9e9a0011c960a6ed9e5466581abf3e3a6c26ee317461add619a",
    "b1ae7f15836cb2286cdd4e2c37bf9bb7da0a2846d06867a429f654b2e7f383c9",
    "9b74f89fa3f93e71ff2c241f32945d877281a6a50a6bf94adac002980aafe5ab",
    "b3a92b5b255019bdaf754875633c2de9fec2ab03e6b8ce669d07cb5b18804638",
    "b5c0b915312b9bdaedd2b86aa2d0f8feffc73a2d37668fd9010179261e25e263",
    "c9d52c5cb1e557b92c84c52e7c4bfbce859408bedffc8a5560fd6e35e10b8800",
    "c555bc5fc3bc096df0a0c9532f07640bfb76bfe4fc1ace214b8b228a1297a4c2",
    "f9dbfafc3af3400954975da24eb325e326960a25b87fffe23eef3e7ed2fb610e",
]
```
暗号学的ハッシュ関数
```
HASH256
```


## 1 親ハッシュを求める
左ハッシュと右ハッシュを連結して親ハッシュを求めます
```
# 左ハッシュ
leftHash = bytes.fromhex('c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5')
# 右ハッシュ
rightHash = bytes.fromhex('c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5')
# 親ハッシュ
parent = hash256(leftHash + rightHash)
print(parent.hex())
```
結果：`8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd`

## 2 マークルペアレントレベルに対応する新しいハッシュリストを求める
５つのハッシュリストを用意すれば末尾のハッシュを複製してハッシュリストを整理した後、ペアを作って連結した結果、要素数３のハッシュリストが出来上がります。
```
# 順序付きハッシュリストをインプットしてハッシュ化
hex_hashes = [
    'c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5',
    'c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5',
    'f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0',
    '3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181',
    '10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae',
]
hashes = [bytes.fromhex(x) for x in hex_hashes]

# ハッシュ数が奇数なら末尾のハッシュ値を複製して連結
if len(hashes) % 2 == 1:
    hashes.append(hashes[-1])

# マークルペアレントレベル
parent_level = []

# ハッシュ値を２つずつ選択してペアを作成
for i in range(0, len(hashes), 2):
    parent = merkle_parent(hashes[i], hashes[i+1])
    parent_level.append(parent)
for item in parent_level:
    print(item.hex())
```
結果
```
8b30c5ba100f6f2e5ad1e2a742e5020491240f8eb514fe97c713c31718ad7ecd
7f4e6f9e224e20fda0ae4c44114237f97cd35aca38d83081c9bfd41feb907800
3ecf6115380c77e8aae56660f5634982ee897351ba906a6837d15ebc3a225df0
```

## 3 マークルルートを求める
常にハッシュリストの長さを1より長いかどうか判定します。最後のハッシュペアが連結されてハッシュリストの長さが1になるとwhileループを終了します。このときに求まるハッシュをマークルルートと呼びます。
```
from helper import merkle_parent_level
hex_hashes = [
    'c117ea8ec828342f4dfb0ad6bd140e03a50720ece40169ee38bdc15d9eb64cf5',
    'c131474164b412e3406696da1ee20ab0fc9bf41c8f05fa8ceea7a08d672d7cc5',
    'f391da6ecfeed1814efae39e7fcb3838ae0b02c02ae7d0a5848a66947c0727b0',
    '3d238a92a94532b946c90e19c49351c763696cff3db400485b813aecb8a13181',
    '10092f2633be5f3ce349bf9ddbde36caa3dd10dfa0ec8106bce23acbff637dae',
    '7d37b3d54fa6a64869084bfd2e831309118b9e833610e6228adacdbd1b4ba161',
    '8118a77e542892fe15ae3fc771a4abfd2f5d5d5997544c3487ac36b5c85170fc',
    'dff6879848c2c9b62fe652720b8df5272093acfaa45a43cdb3696fe2466a3877',
    'b825c0745f46ac58f7d3759e6dc535a1fec7820377f24d4c2c6ad2cc55c0cb59',
    '95513952a04bd8992721e9b7e2937f1c04ba31e0469fbe615a78197f68f52b7c',
    '2e6d722e5e4dbdf2447ddecc9f7dabb8e299bae921c99ad5b0184cd9eb8e5908',
    'b13a750047bc0bdceb2473e5fe488c2596d7a7124b4e716fdd29b046ef99bbf0',
]
hashes = [bytes.fromhex(x) for x in hex_hashes]
current_hashes = hashes
while len(current_hashes) > 1:
    current_hashes = merkle_parent_level(current_hashes)
print(current_hashes[0].hex())
```


## 4


# おまけ
## ハッシュツリーで何を保証できるのか？






これを解決するのがハッシュツリー


## なぜハッシュ同士を連結して親ハッシュを求めるの？


# 感想