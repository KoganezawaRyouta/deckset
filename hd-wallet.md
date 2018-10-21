## 階層的決定性ウォレットの実装

```
m / purpose' / coin_type' / account' / change / address_index
```

![top](https://cdn-images-1.medium.com/max/1600/1*tBfwTsPiwvvMDEmrLMq5fA.png)

---

# Abount Me
  * Twitter ＠kogane5513
  * 2018/10〜　マネーフォワードフィナンシャル
    * 仮想通貨取引所の新規開設に向けて奮闘中！
  * 〜2018/9
    * （株）SmartDrive
      * 新規事業開発
    * freee 株式会社（フリー）
      * 決済、認証基盤チーム

---

## 本日の内容

* 階層的決定性ウォレットとは
* 階層的決定性ウォレットの実装
* MultiSig Addressの実装

この記事では、シード・フレーズを使い、階層的決定性ウォレット/MultiSig Addressを生成するところまでを簡単に説明したいと思いいます。
また、Golangの [btcsuite](https://cdn-images-1.medium.com/max/1600/1*tBfwTsPiwvvMDEmrLMq5fA.png) ライブラリーを使用予定です。

---

## 階層的決定性ウォレットとは

マスターなるシード値から、m/i/0/kのような階層構造的に秘密鍵を生成・管理できる標準規格で、英語では Hierarchical Deterministic Wallet(HD Wallet)と言います。
通常のウォレットでは、使用済みの秘密鍵と公開鍵のペアを定期的に、すべてバックアップしておく必要がありましたが、BIP32で標準化されたプロトコルを利用することで、マスターシードさえ保存されていれば、そこから派生する秘密鍵を持っていなくてもインデックスを指定することで、異なるシステムからでもいつでも再生成し、利用することができます。
ビットコインコアにもv0.13.0から導入され、正式にサポートされるようになりました。

つまり、HD Walletの規格で生成された秘密鍵であれば、Seedさえあれば復元、再利用可能となり運用が楽になるというもの。
下記は、階層化されたパスのイメージです。

>> マスター / 仕様(BIP)' / 通貨' / アカウント' / 支払い or お釣り / アドレス

---

## 階層的決定性ウォレットの実装(シード 生成)

基本的にHD Walletの実装では、hdkeychainパッケージに用意されています。
シードの生成もこのパッケージにあるGenerateSeedファンクションを使えばよくて、
シードの長さはBIP-0032で推奨されているバイト単位の長さ(256bits)も定数として定義されているので、それを使いましょう

```golang

// シード 生成
import (
	"github.com/btcsuite/btcutil/hdkeychain"
)

seed, err := hdkeychain.GenerateSeed(hdkeychain.RecommendedSeedLen)
if err != nil {
   return err
}
```

---

## 階層的決定性ウォレットの実装(マスター鍵)

シードを利用してマスター鍵を生成します。
また、テストネットの場合、第二引数に渡すものはchaincfgパッケージに用意されているchaincfg.TestNet3Paramsをセットしてください。

```golang

import (
  "github.com/btcsuite/btcd/chaincfg"
	"github.com/btcsuite/btcutil/hdkeychain"
)

master, err := hdkeychain.NewMaster(seed, &chaincfg.MainNetParams)
if err != nil {
   return err
}
master.ECPrivKey() // マスターの秘密鍵
```

---

## 階層的決定性ウォレットの実装(Purpose)

ここではどの仕様(BIP)に従っているかを示す。また、強化鍵導出のためインデックス値に2³¹を加算する

```golang

import (
	"github.com/btcsuite/btcutil/hdkeychain"
)

// m/49'
purpose, err := master.Child(49 + hdkeychain.HardenedKeyStart)
if err != nil {
   return err
}
```

---

## 階層的決定性ウォレットの実装(CoinType)

ここではどの通貨に従っているかを示す。また、強化鍵導出のためインデックス値に2³¹を加算する。Coin Type のインデックスについては、こちらに記載されている

```golang

import (
	"github.com/btcsuite/btcutil/hdkeychain"
)

// m/49'/1'
coinType, err := purpose.Child(1 + hdkeychain.HardenedKeyStart)
if err != nil {
   return err
}
```

--

## 階層的決定性ウォレットの実装(Account)

ここではアカウントを示すインデックスを指定する。また、強化鍵導出のためインデックス値に2³¹を加算する

```golang

import (
	"github.com/btcsuite/btcutil/hdkeychain"
)

// m/49'/1'/0'
account, err := coinType.Child(1 + hdkeychain.HardenedKeyStart)
if err != nil {
   return err
}
```

--

## 階層的決定性ウォレットの実装(エクスターナル/インターナル)

ここでは支払い=0、おつり用=1のいづれかを示す

```golang

import (
	"github.com/btcsuite/btcutil/hdkeychain"
)

// m/49'/1'/0'/0
change, err := account.Child(0)
if err != nil {
   return err
}
```

--

## 階層的決定性ウォレットの実装(アドレス)

ここが末端となりここで生成された公開鍵を使ってアドレスを生成する、次回はこの公開鍵利用し P2SH アドレスを生成するところを纏める

```golang

import (
	"github.com/btcsuite/btcutil/hdkeychain"
)

// m/49'/1'/0'/0/0
addressIndex, err := change.Child(0)
if err != nil {
   return err
}
```

--
