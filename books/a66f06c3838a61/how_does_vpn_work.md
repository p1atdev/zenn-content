---
title: "VPN の仕組み"
---

Network Extension のフレームワークに飛び込む前に、VPN やネットワーキング全般に関する理論を少し知っておく必要があります。もしあなたがすでにネットワークや VPN に精通しているのであれば、この記事は読み飛ばしていただいてかまいません。そうでない場合は、この記事でネットワークプロトコルの簡単な概要と再確認を行い、VPN が解決しようとする問題の概要を説明し、既存の VPN プロトコルの一つを分析します。

:::message

**前提条件**
この記事を読むための前提条件はありませんが、ネットワーク・プロトコルの一般的な知識があれば、プラスに働くことでしょう。この記事では、[Wireshark](https://www.wireshark.org/) を多用しています。もしこのツールに馴染みがなければ、インストールして（無料です）、1 つか 2 つの[サンプルキャプチャ](https://wiki.wireshark.org/SampleCaptures)を検査することをお勧めします。

:::

## ネットワーク

ブラウザに URL を打ち込んだときに何が起きているのでしょうか？[Wireshark](https://www.wireshark.org/)で見てみましょう。

### DNS, HTTP (アプリケーション層)

まず、ブラウザは DNS リクエストを送り、与えられたドメイン名(例えば、`www.example.com`など)の IP アドレスを解決する必要があります。

**DNS リクエスト**
ブラウザはドメイン名 `www.example.com` を指定して DNS クエリを送ります。
![](/images/dns_request.png =600x)

**DNS レスポンス**
DNS サーバーが _答え_ を返しました。 - `93.184.216.34`
![](/images/dns_response.png =600x)

[ドメインネームシステム（DNS）](https://www.cloudflare.com/ja-jp/learning/dns/what-is-dns/) は、インターネット上の電話帳のようなものです。人間は、`apple.com` や `nshipster.com` などのドメイン名を通して情報にアクセスします。ブラウザは、IP アドレスを通じて対話します。DNS は、ドメイン名を IP アドレスに変換します[^1]。

[^1]: あなたは IPv4 アドレスを覚えることはできると思いますが、試しに 2600:cb00:2048:1::c629:d7a2（IPv6）を暗記することはできるでしょうか？

:::message

ブラウザが `www.example.com` にリクエストを送るには、IP アドレスを知っている必要があります。どのようにして DNS サーバーにリクエストを送ることができたのでしょうか？通常、DNS サーバーの IP アドレス（または複数のアドレス）は、ルーターに接続する際に DHCP によってコンピューターに提供されます。

:::

これでブラウザの IP アドレスが決まり、`www.example.com (93.184.216.34)` に HTTP リクエストを送り始める準備ができました。

**HTTP リクエスト**

ブラウザは解決された IP アドレスを受け取り、ルートパス（ `/` ）にあるページの内容を取得するために HTTP リクエストを開始します。
![](/images/http_request.png =600x)

**HTTP レスポンス**

サーバーは HTTP Body 内でいくつかの HTML コンテンツを返します。

![](/images/http_response.png =600x)

ここで、アプリケーション層プロトコルの話に多くの時間を割くことにしましょう。HTTP と DNS については、すでにご存じだと思います。この 2 つを例に挙げたのは、HTTP は通常 TCP[^2]の上で、DNS は UDP[^3] の上で動作するからです。

[^2]: HTTP/3 は TCP を UDP をベースにした新しいトランスポート層プロトコルである、 QUIC で置き換えるように見えます。
[^3]: それは必ずしもそうではなく、ブラウザによって異なります。例えば、Firefox はデフォルトで DNS over HTTPS を使用するようになりました。

:::message

もっと詳しく知りたい場合は、MDN に [HTTP に関する素晴らしいセクション](https://developer.mozilla.org/ja/docs/Web/HTTP)がありますし、Cloudflare には [DNS に関するセクション](https://www.cloudflare.com/ja-jp/learning/dns/what-is-dns/)もあります。もう少し深く掘り下げたい場合は、RFC を読むことをお勧めします。[HTTP](https://ja.wikipedia.org/wiki/Hypertext_Transfer_Protocol) と DNS の [RFC のリスト](https://ja.wikipedia.org/wiki/Request_for_Comments#RFC%E3%81%AE%E4%B8%80%E8%A6%A7)は、それぞれの Wiki ページにあります。

:::

### UDP, TCP (トランスポート層)

HTTP と DNS は、特定のアプリケーションレベルの問題を解決します。しかし、コンピュータがメッセージを受信したとき、どのアプリケーションにそのメッセージを送ればいいのか、どうやって知ることができるのでしょうか？そこで登場するのが、[UDP（User Datagram Protocol)](https://ja.wikipedia.org/wiki/User_Datagram_Protocol) や [TCP（Transmission Control Protocol)](https://ja.wikipedia.org/wiki/Transmission_Control_Protocol) などのトランスポート層プロトコルです。

サーバーは通常、起動時に IP アドレスとポート番号（0 ～ 65535 の 16 ビット符号なし整数）を組み合わせて内部エンドポイントであるソケットを作成し、このソケットに到着するメッセージの待ち受けを開始します。

ブラウザがメッセージを送信する必要がある場合もソケットが必要であり、システム API のいずれかを使用してソケットを作成します。

:::message

Apple のプラットフォームでは、Unix ソケット、[`CFSocket`](https://developer.apple.com/documentation/corefoundation/cfsocket-rg7) ラッパー、そして最近導入された [`Network` フレームワーク](https://developer.apple.com/documentation/network/nwconnection)に至るまで、ポート/ソケットを扱うための様々な API が存在します。

:::

TCP や UDP などのトランスポート層プロトコルは、プロトコルデータユニットを用いてデータを転送します。TCP の場合、PDU は[セグメント](https://ja.wikipedia.org/wiki/Transmission_Control_Protocol#TCP%E3%82%BB%E3%82%B0%E3%83%A1%E3%83%B3%E3%83%88%E6%A7%8B%E9%80%A0)であり、UDP の場合は[データグラム](https://ja.wikipedia.org/wiki/User_Datagram_Protocol#%E3%83%91%E3%82%B1%E3%83%83%E3%83%88%E6%A7%8B%E9%80%A0)です。どちらのプロトコルも、送信元と送信先のポート番号を記録するためのヘッダーフィールドを使用します。

DNS メッセージは通常、[UDP](https://datatracker.ietf.org/doc/html/rfc768) を使用して送信されます（他のオプションもあり、[DNS over HTTPS](https://ja.wikipedia.org/wiki/DNS_over_HTTPS) のようなものも含まれます）。UDP データグラムヘッダーは非常にシンプルです。 システムが DNS メッセージを送信する必要がある場合、メッセージ（メッセージのプロトコルは関係なく、DNS では単なる生のバイトです）を受け取り、その前に 4 バイトの UDP データグラムヘッダーを追加します。ブラウザの例に戻ると、UDP ヘッダーは次のようになります。

![](/images/udp_header.png =600x)

*送信元ポート*は `63802` で、これはブラウザが自分用に予約したポートであり、送信先ポートは `53` で、これは[ウェルノウンポート](https://ja.wikipedia.org/wiki/TCP%E3%82%84UDP%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E3%83%9D%E3%83%BC%E3%83%88%E7%95%AA%E5%8F%B7%E3%81%AE%E4%B8%80%E8%A6%A7)の 1 つです。

明らかに、UDP と TCP は全く異なるものです。TCP プロトコルは独自の[セグメント構造](https://ja.wikipedia.org/wiki/Transmission_Control_Protocol#TCP%E3%82%BB%E3%82%B0%E3%83%A1%E3%83%B3%E3%83%88%E6%A7%8B%E9%80%A0)を持っています。TCP はコネクション指向で、オクテットのストリームを、信頼性が高く、順序よく、エラーチェックされた状態で配信します。一方、UDP はシンプルなコネクションレス型の通信モデルで、非常にミニマルなものです。

### IP (インターネット層)

前節で、ソケットは IP アドレスとポート番号の組み合わせでアドレス指定できることを確認しました。ポートが何のためにあるかは分かっています。しかし、UDP/TCP ヘッダには、IP アドレスは見当たりません。どのコンピュータにメッセージを送るかを知るには、IP アドレスと IP プロトコルが必要です。Wireshark のキャプチャに戻り、IP ヘッダーにどんな情報が含まれているかを見てみましょう。

![](/images/ip_header.png =600x)

Wireshark でキャプチャした情報は、すべて私の `en0` _ネットワークインターフェース_（WiFi）から記録されたものです。

:::message

**ネットワークインターフェイス**とは、コンピュータとネットワークを接続するポイントです。コンピューターは複数のネットワークインターフェースを持つことができます。Wi-Fi、Ethernet などです。

`en0` (macOS の WiFi モジュール)のような*物理的なネットワークインターフェイス*があります。`networksetup -listallhardwareports` コマンドを使えば、それらのすべてをリストアップできます。各物理インターフェースには、関連するイーサネットアドレスとハードウェアデバイスがあり、例えば以下のようになります。

```
❯ networksetup -listallhardwareports

Hardware Port: Wi-Fi
Device: en0
Ethernet Address: 3c:22:fb:42:13:71
...
```

そして `lo0` (コンピュータが自分自身と通信するために使う[ループバック](https://askubuntu.com/questions/247625/what-is-the-loopback-device-and-how-do-i-use-it)インターフェース) や `utun` (通常 VPN で使われる。これについては後で詳しく説明します) などの*仮想ネットワー*クインターフェースがあります。`ifconfig` を使って、物理的、仮想的を含む全てのネットワークインターフェイスを見ることができます。

```
❯ ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> mtu 16384
	options=1203<RXCSUM,TXCSUM,TXSTATUS,SW_TIMESTAMP>
	inet 127.0.0.1 netmask 0xff000000
...
```

:::

このセクションのキャプチャで見たすべての情報は、私の [ISP](https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80) と、私と送信先との間のサーバーにも表示されます。彼らは私のすべての DNS クエリーを見たり、私の IP アドレスを見たり、HTTP（非暗号化）の場合、私のすべての HTTP 通信も見たりしています[^4]。この情報を使えば、誰でも私を特定し、追跡することができます。そして、VPN はこのような問題を解決するために設計されているのです。

[^4]: HTTPS を使用している場合は、一般的に問題ありません。HTTP ヘッダとリクエストの本文を読むことができるのは、送信先のサーバーだけです。

## VPN

VPN (Virtual Private Network) とは何でしょうか？VPN はもともと、安全でない公衆網を使用して、コンピュータと保護された企業ネットワークまたはプライベートネットワークとの間に安全な接続を確立するために設計されたものです。そのため、あるコンピュータをプライベート・ネットワークの一部に「仕立て」、そのネットワーク内のコンピュータがそのコンピュータをアドレス指定できるようにすることが一般的でした。

近年、VPN は、オンライン接続時に安全性と匿名性を確保し、トラフィックの送信元を隠して無制限のネットワークアクセスを楽しむ方法として、消費者市場でも爆発的な人気を博しています。ビジネス用 VPN と個人用 VPN は、通常、全く異なる製品です。このシリーズでは、主に個人向け VPN に焦点を当てます。個人向け VPN は、扱うべき複雑さが若干少ないからです。

### VPN の仕組み

VPN のキーとなる考え方は、意外とシンプルです。

> 1.  システムから送られてくるすべての IP パケット[^5] を受け取り
> 2.  それらを暗号化する
> 3.  VPN パケットに暗号化されたデータブロブ(Blob)をカプセル化する
> 4.  VPN パケットを VPN サーバーに送信 (UDP/TCP 経由)

IP パケットは結局、別の IP パケットに包まれ、マトリョーシカをネットワークで再現したような状態になっています。

![](https://kean.blog/images/posts/vpn-protocol/vpn-matryoshka.png =240x)

これはシンプルなアイデアでありながら、私たちの目標をすべて達成するものです。オリジナルの IP パケットは完全に暗号化されています。アプリケーションも、インターネット層のコンテンツ（IP アドレス）さえも見ることができません。ISP や他の監視者が見ることができるのは、VPN クライアントと VPN サーバーの間を行き来する暗号化されたデータの塊だけです。

さて、このようなものをどのように実装するのでしょうか？これについては、次のセクションで取り上げています。IP パケットの直接操作、暗号化、仮想ネットワークインターフェースなどが含まれています。次のセクションに行くのが待ち遠しいですね。

### 例: OpenVPN

VPN の背後にある考え方は単純ですが、プロトコルの設計と実装は簡単とは言い難いものです。独自の VPN プロトコルを設計する前に、[OpenVPN](https://openvpn.net/) のような既存の VPN プロトコルを調査する価値はあります。

https://openvpn.net/

この[プロトコルの説明](https://openvpn.net/community-resources/openvpn-protocol/)は驚くほど短く、シンプルです。最初のステップとして、クライアントはサーバとの**セッションを確立**し、暗号情報を交換します。

**クライアントのリセット**

クライアントは、新しいセッションを準備する `P_CONTROL_HARD_RESET_CLIENT_V2` タイプの*制御*パケット（`P_CONTROL`）を送信します。

![](/images/openvpn_reset_client.png =600x)

**サーバーのリセット**

サーバは自身の制御パケット（`P_CONTROL_HARD_RESET_SERVER_V2` タイプ）で応答します。

![](/images/openvpn_reset_server.png =600x)

**確認**

クライアントは、サーバから「リセット」を受け取ったことを確認します。

![](/images/openvpn_acknowledge.png =600x)

セッションの準備ができたら、クライアントとサーバーは鍵に合意するために暗号データを交換する必要があります。この交換は、TLS[^6] では「ハンドシェイク」と呼ばれています。

[^6]: OpenVPN は、静的キー（事前共有キーを使用）と TLS（認証とキー交換に SSL/TLS と証明書を使用）の 2 つのモードをサポートしています。この例では、後者が表示されています。

**制御メッセージ**

クライアントは、今度はペイロードを持つ別の制御（`P_CONTROL`）パケットを送信します。

![](https://kean.blog/images/posts/vpn-protocol/openvpn-04.png =600x)

**TLS ペイロード**

このパケットは TLS ペイロードを持っています。

![](https://kean.blog/images/posts/vpn-protocol/openvpn-05.png =600x)

TLS の交換に関して詳しく説明するつもりはありません。Wireshark のセッションは、TLS の「ハンドシェイク」の最初のメッセージだけを表示します。ハンドシェイク全体では、数回のやりとりが必要です。ポイントは、どのバージョンの TLS が使われ、どの暗号化モデルが使われたかに関わらず、いくつかの制御メッセージの後、クライアントとサーバーはメッセージを暗号化して送信する準備が整うということです。そしてこれが `P_DATA_V1` メッセージタイプの目的である、実際のトンネルデータの暗号文を含むデータチャネルパケットです。

![](https://kean.blog/images/posts/vpn-protocol/openvpn-06.png =600x)

そして、これで OpenVPN の概要がまとまりました。完全に理解する必要はありませんが、大枠は明確になります。主なポイントは以下の通りです。

-   OpenVPN は、制御 (`P_CONTROL_V1`)、確認応答 (`P_ACK_V1`)、データ (`P_DATA_V1`) という複数のメッセージタイプを持つバイナリプロトコル
-   クライアントとサーバーがデータ (`P_DATA_V1`) の交換を開始する前に、セッションを確立して暗号情報を交換する必要がある
-   メッセージは UDP[^7] で送信される。制御メッセージ (`P_CONTROL_V1`) は配送保証があり、確認応答 (`P_ACK_V1`) が必要である。データメッセージ (`P_DATA_V1`) は確認応答を必要とせず、配送保証もない。元のメッセージの基本プロトコルは、必要な保証を提供する責任がある。

[^7]: OpenVPN はデフォルトで UDP を使用しますが、TCP もサポートしています。

これは、私たちのおもちゃの VPN プロトコルの作業を開始するのに十分な情報です。

### 参考文献(原文)

1. Wireshark, [OpenVPN Sample Capture](https://wiki.wireshark.org/OpenVPN)
2. Cloudflare, [What is DNS?](https://www.cloudflare.com/learning/dns/what-is-dns/)
3. Mozilla, [HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
4. OpenVPN, [OpenVPN Protocol](https://openvpn.net/community-resources/openvpn-protocol/)

### 翻訳する際に参考にした文献

https://ja.wikipedia.org/wiki/Hypertext_Transfer_Protocol
https://ja.wikipedia.org/wiki/User_Datagram_Protocol
https://ja.wikipedia.org/wiki/Transmission_Control_Protocol
https://ja.wikipedia.org/wiki/Request_for_Comments
https://ja.wikipedia.org/wiki/TCP%E3%82%84UDP%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8B%E3%83%9D%E3%83%BC%E3%83%88%E7%95%AA%E5%8F%B7%E3%81%AE%E4%B8%80%E8%A6%A7
https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%83%97%E3%83%AD%E3%83%90%E3%82%A4%E3%83%80

## 原文

https://kean.blog/post/networking-101
