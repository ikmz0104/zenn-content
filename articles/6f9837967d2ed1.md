---
title: "ECDSAでトランザクションの署名を行う"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Node.js", "Solidity"]
published: false
---

# ECDSAとは
ECDSA（Elliptic Curve Digital Signature Algorithm）とは楕円曲線デジタル署名アルゴリズムと呼ばれる暗号技術の１種で、ブロックチェーンを用いたネットっワーク上では仮想通貨の取引が正しく行われているかを検証するために用いられます。

例えば、イーサリアムネットワーク上でアリス（購入者）とボブ（出品者）がこのような取引をするとします。
- ボブさんは絵画を中心としたアーティストでありイーサリアムネットワーク上に複数のNFTを出品している
- アリスさんはボブさんのNFTを購入しようとし、1ETHをボブさんに送金した

上記のような取引をする際に最も重要なのが、`そのトランザクションは誰が作成したものであり、誰が証明できるのか`という点です。

こういった場合に用いられるのがECDSAです。

今回は`トランザクションの署名はどのように作成されるのか`、`どのように署名を検証しているのか`の２点に絞って解説します。

※楕円曲線に関する解説は行いません。

# 実装手順
- 検証したいメッセージ、秘密鍵を用意
- `EthUtil.ecsign`メソッドで署名を作成
- `EthUtil.ecrecover`メソッドで署名から署名者の公開鍵を導出
- `EthUtil.toChecksumAddress`メソッドで導出アドレスと実際の署名者のアドレスを比較

# 環境
WSL2

Ubuntu 20.04 on Windows(11)

#　プロジェクトのセットアップ
プロジェクト階層構造
```
├── node_modules
├── .gitignore
├── index.js
├── package-lock.json
├── package.json
├── usage.js
└── yarn.lock
```

`.env`ファイル
```
SIGNED_ACOUNT_ADDRESS_ON_ETHERIUM = 0x89c24a88bad4abe0a4f5b2eb5a86db1fb323832c  //署名者のアドレス
PRIVATE_KEY = 0x61ce8b95ca5fd6f55cd97ac60817777bdf64f1670e903758ce53efc32c3dffeb  //署名者が保有する秘密鍵
```

利用するモジュール
```js: index.js
const EthUtil = require('ethereumjs-util')
require('dotenv').config();
const env = process.env
```

# トランザクションの署名はどのように作成されるのか
`MESSAGE`はMetaMaskを利用していればよく見るモーダル画面のメッセージ部分のテキスト内容です。いつ署名したのかという点も重要な情報になるので`timestamp`を取り入れています（本来はtimestampを用いた時間的制御も必要ですが今回はおまけで書いています）。

次に`EthUtil`モジュールを用いて変数をハッシュ化し`HASHED_MESSAGE`と`HASHED_PRIVATE_KEY`を得ます。

単純なメッセージとバッファ化されたメッセージを`console.log`で比較します。
```
ようこそ!
Address: 0x89c24a88bad4abe0a4f5b2eb5a86db1fb323832c
timestamp: 1647030437067

<Buffer 14 fd a9 19 6b b7 9b d7 83 54 2c be 85 2a 2e 94 ac 3c 80 28 fd 81 46 2b 94 48 e8 e2 0b 26 b3 24>
```
```js: index.js
const SIGNED_ACOUNT_ADDRESS_ON_ETHERIUM = env.SIGNED_ACOUNT_ADDRESS_ON_ETHERIUM;
const PRIVATE_KEY = env.PRIVATE_KEY;

const timestamp = Date.now();
const MESSAGE = 'ようこそ!\n' + 'Address: ' + SIGNED_ACOUNT_ADDRESS_ON_ETHERIUM + '\n' + 'timestamp: ' + timestamp;
const HASHED_MESSAGE = EthUtil.keccakFromString(MESSAGE)
const HASHED_PRIVATE_KEY = EthUtil.toBuffer(PRIVATE_KEY)
```

最後に、`EthUtil.ecsign`メソッドの引数に`HASHED_MESSAGE`と`HASHED_PRIVATE_KEY`をわたして署名を生成します（API叩くだけの質素な実装）。
```js: index.js
//メッセージと秘密鍵から署名を作成する関数
function getSignature(hashedMessage, privateKey) {
  const createdSignature = EthUtil.ecsign(hashedMessage, privateKey);
  return createdSignature;
}

...

//署名を作成する
const CREATED_SIGNATURE = getSignature(HASHED_MESSAGE, HASHED_PRIVATE_KEY);
```

作成した署名を`console.log(CREATED_SIGNATURE);`で確認します。
```
{
  r: <Buffer 1f 69 ab 34 5f 4b 93 30 fc 4d 9c f4 4e 36 f8 b4 af 2c 0d c7 75 d9 99 f7 38 58 6e 39 95 56 68 de>,
  s: <Buffer 73 cb bc 6d 34 10 e3 c3 9d b0 cb 78 2f 67 53 ca b9 69 d3 f2 ff d9 99 d1 00 ae c8 52 79 93 71 6b>,
  v: 28
}
```

## 実装上注意する点
メッセージのハッシュ化を行い際に単純にバッファをかけるだけではなく、用途に応じた関数を指定しなくてはいけない。
例えば`EthUtil.toBuffer`メソッドはプレフィックスが`0x`のもしかサポートしていないメソッドなので、単純な文字列をバッファ化させるようにメソッドを選定する。

このように`EthUtil.toBuffer`メソッドでは文字列をバッファ化できない
```js: index.js
const MESSAGE = 'honyohonyo'
const HASHED_MESSAGE = EthUtil.toBuffer(MESSAGE)
```
```
Error: This method only supports Buffer but input was: honyohonyo
at assertIsBuffer (/home/ikmz/dev/ECDSA/node_modules/ethereumjs-util/dist/helpers.js:23:15)
at Object.keccak (/home/ikmz/dev/ECDSA/node_modules/ethereumjs-util/dist/hash.js:15:34)
at sign (/home/ikmz/dev/ECDSA/index.js:27:33)
at Object. (/home/ikmz/dev/ECDSA/index.js:40:19)
```

モジュールは全てここに記載されているので適度にググったり翻訳して地道に探そう！
https://github.com/ethereumjs/ethereumjs-util/tree/master/docs


# どのように署名を検証しているのか
[EthUtil.ecrecover](https://github.com/ethereumjs/ethereumjs-util/blob/master/docs/modules/_signature_.md#const-ecrecover)メソッドで先ほど入手した署名をもとに公開鍵を導出します。

引数には、ハッシュ化された署名メッセージ、バッファ化された署名の`r`、バッファ化された署名の`s`、ブロックチェーンネットワークIDを用いる仕様です。署名検証に用いる`EthUtil.toChecksumAddress`メソッドにて「警告：chainIdがある場合とない場合のチェックサムは異なります」とあるので、今回は`EthUtil.ecrecover`メソッドのブロックチェーンネットワークIDを指定しません。

```
<Buffer ff f4 9b 58 b8 31 04 ff 16 87 54 52 85 24 66 a4 6c 71 69 ba 4e 36 8d 11 83 0c 91 70 62 4e 0a 95 09 08 0a 05 a3 8c 18 84 17 18 ea 4f c1 34 83 ac 46 7d ... 14 more bytes>
```

次に、`EthUtil.pubToAddress`メソッドで引数に渡した公開鍵のイーサリアムアドレスを返し、`EthUtil.bufferToHex`メソッドでバッファを`0x`プレフィックス付きの16進文字列に変換します。


最後に、`EthUtil.ecrecover`メソッドで署名の検証を行います。


```js: index.js
//メッセージと署名をもとに、署名者の Ethereum アドレスを導出し署名者が正しいかどうかを返すメソッド
function getVerifiedSigner(hashedMessage, createdSignature, signedAccountAddress) {
  //作成された署名から公開鍵を導出
  const publicKey = EthUtil.ecrecover(hashedMessage, createdSignature.v, EthUtil.toBuffer(createdSignature.r), EthUtil.toBuffer(createdSignature.s));
  //公開鍵から署名者のアドレスを導出
  const signerAccountAddress = EthUtil.bufferToHex(EthUtil.pubToAddress(publicKey));

  //導出したアドレスと署名時のアドレスを比較する
  if(EthUtil.toChecksumAddress(signerAccountAddress) == EthUtil.toChecksumAddress(signedAccountAddress)){
    return true;
  }else{
    return false;
  }
}

//署名を作成する
const CREATED_SIGNATURE = getSignature(HASHED_MESSAGE, HASHED_PRIVATE_KEY);

//署名を検証する
const isVerified = getVerifiedSigner(HASHED_MESSAGE, CREATED_SIGNATURE, SIGNED_ACOUNT_ADDRESS_ON_ETHERIUM)
console.log(isVerified)
```

それでは署名の検証を行います。

もし導出されたアドレスと署名時に用いたアドレスが等しければ`true`、そうでなければ`false`が返るはずです。
```
ikmz@ikmz:~/dev/ECDSA$ node index.js
true
```

OK！

## 実装上必要な知識
公開鍵から署名者のアドレスを導出する意味ですが、公開鍵暗号方式に関しては一般にこのような内容です。
- 公開鍵：サーバ側が持ってるもの（鍵穴）
- 秘密鍵：所有者が持つもの（鍵）
なので「鍵穴を見たら、鍵を持ってる人間の情報を導出できるってことかな」というような理解でOKだと思います。

もちろんハッシュの衝突性から秘密鍵の導出まではできません。

また、仮にウォレットアドレスの秘密鍵が流出した場合、秘密鍵所有者が所有しているNFTやメインネットの仮想通貨を操作できるようになりますので保管には十分気を付けてください。


# おまけ
署名の際に表示されるメッセージが文字化けする事象が発生するらしい。
https://github.com/MetaMask/metamask-extension/issues/5473
https://github.com/MetaMask/metamask-extension/issues/3931

# 感想
業務でブロックチェーン処理を書くことはないですが、ブロックチェーン処理に関わる処理やデバッグ作業を行うことは多いので今回は署名アルゴリズムの一つであるECDSAを抜粋して書いてみました。まだまさブロックチェーン領域は未知な点が多いですが出会った用語をなるべくとりこぼさず実装までこぎつけていければと思います。最後までお読みいただきありがとうございました。