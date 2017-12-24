---
layout: post
title: "gethのコードを追ってトランザクションの理解を深める"
description: ""
date: 2017-12-24
tags: ["ethereum", "geth"]
comments: true
share: true
---

この記事は [Ethereum Advent Calendar 2017](https://qiita.com/advent-calendar/2017/ethereum) の24日目の記事です。


Ethereumのトランザクションへの理解を深めるために[go-ethereum](https://github.com/ethereum/go-ethereum)のコードを軽く読んでみました。  
実際に読んだのはJSON-RPCで[eth_sendTransaction](https://github.com/ethereum/wiki/wiki/JSON-RPC#eth_sendtransaction)を呼び出した際に内部で実行されるコードで、以下のものがそれに該当するメソッドとなります。

[/internal/ethapi/api.go#L1141,L1176](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/internal/ethapi/api.go#L1141,L1176)
```golang
// SendTransaction creates a transaction for the given argument, sign it and submit it to the
// transaction pool.
func (s *PublicTransactionPoolAPI) SendTransaction(ctx context.Context, args SendTxArgs) (common.Hash, error) {

	// Look up the wallet containing the requested signer
	account := accounts.Account{Address: args.From}

	wallet, err := s.b.AccountManager().Find(account)
	if err != nil {
		return common.Hash{}, err
	}

	if args.Nonce == nil {
		// Hold the addresse's mutex around signing to prevent concurrent assignment of
		// the same nonce to multiple accounts.
		s.nonceLock.LockAddr(args.From)
		defer s.nonceLock.UnlockAddr(args.From)
	}

	// Set some sanity defaults and terminate on failure
	if err := args.setDefaults(ctx, s.b); err != nil {
		return common.Hash{}, err
	}
	// Assemble the transaction and sign with the wallet
	tx := args.toTransaction()

	var chainID *big.Int
	if config := s.b.ChainConfig(); config.IsEIP155(s.b.CurrentBlock().Number()) {
		chainID = config.ChainId
	}
	signed, err := wallet.SignTx(account, tx, chainID)
	if err != nil {
		return common.Hash{}, err
	}
	return submitTransaction(ctx, s.b, signed)
}
```

第2引数の`SendTxArgs`は以下のstructとなっていて、APIを叩く際に指定したパラメータが入ったものが渡されます。

[/internal/ethapi/api.go#L1067,L1079](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/internal/ethapi/api.go#L1067,L1079)
```golang
// SendTxArgs represents the arguments to sumbit a new transaction into the transaction pool.
type SendTxArgs struct {
	From     common.Address  `json:"from"`
	To       *common.Address `json:"to"`
	Gas      *hexutil.Big    `json:"gas"`
	GasPrice *hexutil.Big    `json:"gasPrice"`
	Value    *hexutil.Big    `json:"value"`
	Nonce    *hexutil.Uint64 `json:"nonce"`
	// We accept "data" and "input" for backwards-compatibility reasons. "input" is the
	// newer name and should be preferred by clients.
	Data  *hexutil.Bytes `json:"data"`
	Input *hexutil.Bytes `json:"input"`
}
```

`Input`というのは最近追加されたもので、`Data`に取って代わるものみたいですね。


それではさっそく`SendTransaction`の中身を上から順に追っていきましょう。



### Wallet取得

```golang
	// Look up the wallet containing the requested signer
	account := accounts.Account{Address: args.From}

	wallet, err := s.b.AccountManager().Find(account)
	if err != nil {
		return common.Hash{}, err
	}
```

まず始めに送信元アドレスから対応する`Wallet`を見つける処理が行われています。



### Nonce排他制御

```golang
	if args.Nonce == nil {
		// Hold the addresse's mutex around signing to prevent concurrent assignment of
		// the same nonce to multiple accounts.
		s.nonceLock.LockAddr(args.From)
		defer s.nonceLock.UnlockAddr(args.From)
	}
```

`args.Nonce`が指定されていない場合に複数のトランザクションで同じNonceが使われないように排他制御を行っています。
ここでのNonceというのはアカウントに紐づく整数であり、アカウントが発行したトランザクションに対応してカウントアップされます。



### パラメータのデフォルト値設定

```golang
	// Set some sanity defaults and terminate on failure
	if err := args.setDefaults(ctx, s.b); err != nil {
		return common.Hash{}, err
	}
```

先程の`SendTxArgs`で値を指定されなかったパラメータをデフォルト値に差し替えています。
`args.setDefaults`の中身は以下の通り。

[/internal/ethapi/api.go#L1081,L1107](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/internal/ethapi/api.go#L1081,L1107)
```golang
// setDefaults is a helper function that fills in default values for unspecified tx fields.
func (args *SendTxArgs) setDefaults(ctx context.Context, b Backend) error {
	if args.Gas == nil {
		args.Gas = (*hexutil.Big)(big.NewInt(defaultGas))
	}
	if args.GasPrice == nil {
		price, err := b.SuggestPrice(ctx)
		if err != nil {
			return err
		}
		args.GasPrice = (*hexutil.Big)(price)
	}
	if args.Value == nil {
		args.Value = new(hexutil.Big)
	}
	if args.Nonce == nil {
		nonce, err := b.GetPoolNonce(ctx, args.From)
		if err != nil {
			return err
		}
		args.Nonce = (*hexutil.Uint64)(&nonce)
	}
	if args.Data != nil && args.Input != nil && !bytes.Equal(*args.Data, *args.Input) {
		return errors.New(`Both "data" and "input" are set and not equal. Please use "input" to pass transaction call data.`)
	}
	return nil
}
```

`Gas`のデフォルト値である`defaultGas`は同じファイル内に定義されていて`defaultGas = 90000`となっています。  
`GasPrice`のデフォルト値に使用されている`b.SuggestPrice(ctx)`はコードを辿ると[/eth/gasprice/gasprice.go#L75,L147](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/eth/gasprice/gasprice.go#L75,L147)に実際の処理が書いてあるので、気になった方はそちらを見てみて下さい。(これ以外にLESの実装もあります)  
`Value`のデフォルト値は`hexutil.Big`のゼロ値のポインタなっていますね。ちなみに`hexutl.Big`とは`math/big`の`Int`に別名を設けたもので、JSONにしたときに16進数として扱われるようにメソッドがいくつか生えています。  
`Nonce`のデフォルト値には`b.GetPoolNonce(ctx, atgs.From)`が使用されていて、送信元アドレスに対応するアカウントのNonceを取得しています。  
`Data`と`Input`の両方が指定されている場合に値が異なるとエラーとなるようです。



### トランザクション作成

```golang
	// Assemble the transaction and sign with the wallet
	tx := args.toTransaction()
```

各パラメータからトランザクションを作成しています。
`args.toTransaction`の中身は以下の通りです。

[/internal/ethapi/api.go#L1109,L1120](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/internal/ethapi/api.go#L1109,L1120)
```golang
func (args *SendTxArgs) toTransaction() *types.Transaction {
	var input []byte
	if args.Data != nil {
		input = *args.Data
	} else if args.Input != nil {
		input = *args.Input
	}
	if args.To == nil {
		return types.NewContractCreation(uint64(*args.Nonce), (*big.Int)(args.Value), (*big.Int)(args.Gas), (*big.Int)(args.GasPrice), input)
	}
	return types.NewTransaction(uint64(*args.Nonce), *args.To, (*big.Int)(args.Value), (*big.Int)(args.Gas), (*big.Int)(args.GasPrice), input)
}
```

送信先アドレスが空の場合はコントラクト生成用のトランザクションが作られていることがわかります。
(キャストが大量発生していてちょっと苦しそう...)  
`types.Transaction`は以下のstructとなっています。

[/core/types/transaction.go#L49,L72](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/core/types/transaction.go#L49,L72)
```golang
type Transaction struct {
	data txdata
	// caches
	hash atomic.Value
	size atomic.Value
	from atomic.Value
}

type txdata struct {
	AccountNonce uint64          `json:"nonce"    gencodec:"required"`
	Price        *big.Int        `json:"gasPrice" gencodec:"required"`
	GasLimit     *big.Int        `json:"gas"      gencodec:"required"`
	Recipient    *common.Address `json:"to"       rlp:"nil"` // nil means contract creation
	Amount       *big.Int        `json:"value"    gencodec:"required"`
	Payload      []byte          `json:"input"    gencodec:"required"`

	// Signature values
	V *big.Int `json:"v" gencodec:"required"`
	R *big.Int `json:"r" gencodec:"required"`
	S *big.Int `json:"s" gencodec:"required"`

	// This is only used when marshaling to JSON.
	Hash *common.Hash `json:"hash" rlp:"-"`
}
```



### chainIDの取得

```golang
	var chainID *big.Int
	if config := s.b.ChainConfig(); config.IsEIP155(s.b.CurrentBlock().Number()) {
		chainID = config.ChainId
	}
```

リプレイ攻撃防ぐために署名時に使用するchainIDの取得をしています。if文では現在のブロックが[EIP155](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md)を含むハードフォークの後のものであるかどうかを判定しています。  



### 署名

```golang
	signed, err := wallet.SignTx(account, tx, chainID)
	if err != nil {
		return common.Hash{}, err
	}
```

パスフレーズと先程のchainIDを用いてトランザクションに署名を行います。  
gethの中には`Wallet`の実装として`keystore`と`usbwallet`の2種類あって、`keystore`ものではバリデーションと以下の処理が行われています。

[/accounts/keystore/keystore.go#L270,L285](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/accounts/keystore/keystore.go#L270,L285)
```golang
// SignTx signs the given transaction with the requested account.
func (ks *KeyStore) SignTx(a accounts.Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error) {
	// Look up the key to sign with and abort if it cannot be found
	ks.mu.RLock()
	defer ks.mu.RUnlock()

	unlockedKey, found := ks.unlocked[a.Address]
	if !found {
		return nil, ErrLocked
	}
	// Depending on the presence of the chain ID, sign with EIP155 or homestead
	if chainID != nil {
		return types.SignTx(tx, types.NewEIP155Signer(chainID), unlockedKey.PrivateKey)
	}
	return types.SignTx(tx, types.HomesteadSigner{}, unlockedKey.PrivateKey)
}
```

アカウントが事前にアンロックされていない場合はここでエラーとなるようです。  
どのように署名が行われるかについては、機会があれば別の記事として上げようと思います。



### トランザクションプールに投入

```golang
	return submitTransaction(ctx, s.b, signed)
```

遂に最後の行まで来ました...  
`submitTransaction`の中身は以下の通り。

[/internal/ethapi/api.go#L1122,L1139](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/internal/ethapi/api.go#L1122,L1139)
```golang
// submitTransaction is a helper function that submits tx to txPool and logs a message.
func submitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error) {
	if err := b.SendTx(ctx, tx); err != nil {
		return common.Hash{}, err
	}
	if tx.To() == nil {
		signer := types.MakeSigner(b.ChainConfig(), b.CurrentBlock().Number())
		from, err := types.Sender(signer, tx)
		if err != nil {
			return common.Hash{}, err
		}
		addr := crypto.CreateAddress(from, tx.Nonce())
		log.Info("Submitted contract creation", "fullhash", tx.Hash().Hex(), "contract", addr.Hex())
	} else {
		log.Info("Submitted transaction", "fullhash", tx.Hash().Hex(), "recipient", tx.To())
	}
	return tx.Hash(), nil
}
```

gethにはトランザクションプール([TxPool](https://github.com/ethereum/go-ethereum/blob/9d187f02389ba12493112c7feb15a83f44e3a3ff/core/tx_pool.go#L186))というものがあって、それがトランザクションの適用の管理などを行っているようです。`b.SendTx(ctx, tx)`の処理で生成したトランザクションをプールに投入しています。  
トランザクションプールでの処理が気になるのですがコード量が多くて今回は読みきれませんでした... これも機会があれば別の記事として上げたいです。  
コントラクト生成用のトランザクションかどうかで途中分岐していますがこれは処理には関係なく、最終的にはトランザクションのハッシュ値が返されます。



## おわりに
Ethereumのトランザクションが作られる過程の一端をコードで追ってみましたが如何でしたでしょうか？ 今回の記事では表面的な部分しかカバー出来ていないので、より深く理解するためには署名の処理やトランザクションプールのコードを読む必要があると思います。  
Ethereumには通貨やICOとしての機能やÐAppの可能性など様々な側面があると思いますが、こうやって核となるクライアントのコードを読んでみると何か面白い発見があるかもしれませんね。  
余談ですが [Ethereum Advent Calendar 2017 の初日の記事](https://qiita.com/amachino/items/605ff76209d7193dc92c)に書いてあるとおり、EthereumコミュニティのSlackチームがあるのでEthereumに興味のある方は是非参加してみるといいと思います。

