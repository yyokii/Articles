---
title: "[Solidity] Derived contract must override function ~ エラーと菱形継承問題"
emoji: "💎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [solidity]
published: true
---

## 問題

OpenZeppelinなどを利用して開発している際に、コントラクトにおいて多重継承を行い且つ継承元が同じfunctionを定義している場合、下記エラーが生じます。

`Derived contract must override function {function名}. Two or more base classes define function with same name and parameter types.`

## 解決方法

例えば、OpenZeppelin利用時にERC721URIStorage, ERC721Enumerableを継承するとエラーが生じますが、重複したものすべてについて下記のように定義するとエラーが解消されます。

```solidity
contract DemoNFT is ERC721URIStorage, ERC721Enumerable {

~

  function tokenURI(uint256 tokenId)
    public
    view
    override(ERC721, ERC721URIStorage) // 両方のインターフェイスをoverrideする
    returns (string memory)
  {
    return super.tokenURI(tokenId); // 親のfunctionを呼び出す
  }

~

}
```

## Diamond problem

今回の場合、ERC721URIStorage, ERC721EnumerableがどちらもERC721を継承しているので、それらを継承しているDemoNFTにてoverrideが必要になっています。

このときの継承の関係は次のようになっています。

```
           ERC721
       /            \
ERC721URIStorage    ERC721Enumerable
       \            /
           DemoNFT
```

ERC721URIStorage, ERC721EnumerableでERC721のfunctionがoverrideされている場合、DemoNFTではそのどちらを呼び出すかにおいて曖昧さが発生します。

これは[**菱形継承問題**（ひしがたけいしょうもんだい、[英] diamond problem](https://ja.wikipedia.org/wiki/%E8%8F%B1%E5%BD%A2%E7%B6%99%E6%89%BF%E5%95%8F%E9%A1%8C)と呼ばれる問題です。

この問題に対するアプローチは複数ありますが、Solidityの場合[C3 Linearization](https://en.wikipedia.org/wiki/C3_linearization)による解決を行なっています。これにより、線型化されて呼び出し順序が明確になります。

また、Solidityではfunctionが複数で定義されている場合は`is`キーワードの右から左に探索します。

> // Contracts can inherit from multiple parent contracts.
> // When a function is called that is defined multiple times in
> // different contracts, parent contracts are searched from
> / right to left, and in depth-first manner.

[Inheritance | Solidity by Example | 0.8.13](https://solidity-by-example.org/inheritance/)

以上より`contract DemoNFT is ERC721URIStorage, ERC721Enumerable { ~`とした場合の解決順序は
DemoNFT→ERC721Enumerable→ERC721URIStorage→ERC721
となり呼び出し順序が一意に決定します。

また、C3 Linearizationを利用しているため`super`を利用した呼び出しが、親ではなく兄弟関係にあるコンストラクトへの呼び出しになることがあります。
[Solidity Diamond Inheritance - Smart Contracts / Guides and Tutorials - OpenZeppelin Forum](https://forum.openzeppelin.com/t/solidity-diamond-inheritance/2694)

## 参考

* [菱形継承問題 - Wikipedia](https://ja.wikipedia.org/wiki/%E8%8F%B1%E5%BD%A2%E7%B6%99%E6%89%BF%E5%95%8F%E9%A1%8C)
* [C3 linearization - Wikipedia](https://en.wikipedia.org/wiki/C3_linearization)
* [Calling Parent Contracts | Solidity by Example | 0.8.13](https://solidity-by-example.org/super/)
* [Inheritance | Solidity by Example | 0.8.13](https://solidity-by-example.org/inheritance/)
* [Solidity Diamond Inheritance - Smart Contracts / Guides and Tutorials - OpenZeppelin Forum](https://forum.openzeppelin.com/t/solidity-diamond-inheritance/2694)
