---
title: "VPN プロトコルを構築してみよう"
---

## VPN

VPN とは何でしょうか？VPN (Virtual Private Network) はもともと、インターネットなどの安全でない公衆網を使用して、コンピュータと保護された企業ネットワークまたはプライベートネットワークとの間に安全な接続を確立するために設計されたものです。近年、VPN は、インターネットを利用する際に安全かつ匿名で利用でき、さらにトラフィックの送信元を隠すことで無制限のネットワークアクセスを楽しむことができる方法として、消費者市場でも人気を博しています。

ちょっと抽象的すぎると思われるかもしれませんが、その通りです。このシリーズでは、VPN を機能させるための断片をほとんど網羅するつもりです。そして、カスタム VPN プロトコルを実装すること以上に良い方法はあるのでしょうか？

:::message

**必要条件**

このシリーズのサンプルコードを実行するには、物理的なデバイスと適切なエンタイトルメント(訳注: AppGroups や Capability などの設定に関するもの)を取得するための Apple Developer Account が必要です。

:::

## Network Extension Framework

iOS が VPN の設定をどのように管理するか、カスタム VPN 設定をプログラムで作成する方法、VPN のクライアント側とサーバー側の両方を実装する方法について学びます。

このシリーズの主なトピックは、[Network Extension](https://developer.apple.com/documentation/networkextension) フレームワークです。その多くの機能の中でも、仮想プライベートネットワーク（VPN）を広範囲にサポートしています。カスタムプロトコル実装のクライアント側は、[Packet Tunnel Provider](https://developer.apple.com/documentation/networkextension/packet_tunnel_provider) 拡張として実装されています。Packet Tunnel Provider 拡張を含むアプリは、NETunnelProviderManager を使用して、カスタムプロトコルを使用する VPN 構成を作成および管理し、その構成によって指定される VPN 接続を制御します。

:::message

アプリの全ソースコードは、[kean/VPN](https://github.com/kean/VPN/) で公開されています。

:::
