---
title: "【Swift5】TableViewなどでセルを長押しした時に出るメニューをContextMenuSwiftでサクッと作る"
emoji: "📲"
type: "tech"
topics: ["swift", "contextmenuswift"]
published: true
---

いろんなアプリでよく見かける、CollectionViewやTableViewなどでセルを長押ししたら出てくるメニューを簡単に作れるライブラリ[ContextMenuSwift](https://github.com/umerjabbar/ContextMenuSwift)の使い方です

# 環境
- macOS Big Sur 11.3
- Xcode 12.4
- Swift 5
- ContextMenuSwift 0.0.8

# 使用するライブラリの情報
### ContextMenuSwift
https://github.com/umerjabbar/ContextMenuSwift

### インストール
以下を`Podfile`に追記
```ruby
pod 'ContextMenuSwift'
```
そして`pod install`

### Githubに載っている例
![](https://storage.googleapis.com/zenn-user-upload/61ki7r9um3nnq3viuy3et7h1v7t9 =248x)
![](https://storage.googleapis.com/zenn-user-upload/ull3i62n3xnmar143qb7w0fdkfws =248x)
![](https://storage.googleapis.com/zenn-user-upload/jknyw3ucayazrwqlnxtrh6jdf2t0 =248x)

# 作る
## インポート
まずは使いたいCollectionViewかTableViewのところでインポートします
今回はTableViewで作っていくので、各自自分のプロジェクトに合わせて置き換えてください
```swift: HogeTableViewController.swift
import ContextMenuSwift
```

## extensionの設定
次に、`extension`で`ContextMenuDelegate`を拡張し、コンテキストメニューが表示された際の挙動を設定します。
```swift: HogeTableViewController.swift
// MARK: ContextMenuDelegate
extension HogeTableViewController: ContextMenuDelegate {
    
    /**
        コンテキストメニューの選択肢が選択された時に実行される
        - Parameters:
            - contextMenu: そのコンテキストメニューだと思われる
            - cell: **選択されたコンテキストメニューの**セル
            - targetView: コンテキストメニューの発生源のビュー
            - item: 選択されたコンテキストのアイテム(タイトルとか画像とかが入ってる)
            - index: **選択されたコンテキストのアイテムの**座標
        - Returns: よくわからない(多分成功したらtrue...?)
     */
    func contextMenuDidSelect(_ contextMenu: ContextMenu,
                              cell: ContextMenuCell,
                              targetedView: UIView,
                              didSelect item: ContextMenuItem,
                              forRowAt index: Int) -> Bool {
        
        print("コンテキストメニューの", index, "番目のセルが選択された！")
	print("そのセルには", item.title, "というテキストが書いてあるよ!")
        
        //サンプルではtrueを返していたのでとりあえずtrueを返してみる
        return true
        
    }
    
    /**
        コンテキストメニューの選択肢が選択された時に実行される
        - Parameters:
            - contextMenu: そのコンテキストメニューだと思われる
            - cell: **選択されたコンテキストメニューの**セル
            - targetView: コンテキストメニューの発生源のビュー
            - item: 選択されたコンテキストのアイテム(タイトルとか画像とかが入ってる)
            - index: **選択されたコンテキストのアイテムの**座標
     こちらは値を返さない方
      (値を返す方との違いがよくわからないが、サンプルでは返す方を使っていたのでそちらを使うことを推奨)
     */
    func contextMenuDidDeselect(_ contextMenu: ContextMenu,
                                cell: ContextMenuCell,
                                targetedView: UIView,
                                didSelect item: ContextMenuItem,
                                forRowAt index: Int) {
    }
    
    /**
     コンテキストメニューが表示されたら呼ばれる
     */
    func contextMenuDidAppear(_ contextMenu: ContextMenu) {
        print("コンテキストメニューが表示された!")
    }
    
    /**
     コンテキストメニューが消えたら呼ばれる
     */
    func contextMenuDidDisappear(_ contextMenu: ContextMenu) {
        print("コンテキストメニューが消えた!")
    }
    
    
}
```

## 長押しされた時の関数を作成
次は、コンテキストメニューの内容を作っていきます
セルに長押しの判定を付与し、長押しされた時にコンテキストメニューを呼び出す形です

以下の関数を作成し、長押しされた際に実行するものを中に入れます
```swift: HogeTableViewController.swift
/// セルが長押しした際に呼ばれるメソッド 
@objc func cellLongPressed(_ recognizer: UILongPressGestureRecognizer) {

    // 押された位置でcellのPathを取得
    let point = recognizer.location(in: tableView)
    // 押された位置に対応するindexPath
    let indexPath = tableView.indexPathForRow(at: point)
        
    if indexPath == nil {  //indexPathがなかったら
            
        return  //すぐに返り、後の処理はしない
            
    } else if recognizer.state == UIGestureRecognizer.State.began  {
        // 長押しされた場合の処理
            
        //コンテキストメニューの内容を作成します
        let edit = ContextMenuItemWithImage(title: "編集", image: UIImage(systemName: "square.and.pencil")!)
        let delete = ContextMenuItemWithImage(title: "削除", image: UIImage(systemName: "trash")!)
            
	 //コンテキストメニューに表示するアイテムを決定します
        CM.items = [edit, delete]
	//表示します
        CM.showMenu(viewTargeted: tableView.cellForRow(at: indexPath!)!,
                    delegate: self,
                    animated: true)
            
    }
}
```

## tableViewにrecognizerを結びつける
次に、これを`viewDidLoad()`でTableViewに追加します
```swift: HogeTableViewController.swift
override func viewDidLoad() {
    super.viewDidLoad()
	
    //〜その他の処理〜
    
    //長押し時の判定
    // UILongPressGestureRecognizer宣言
    let longPressRecognizer = UILongPressGestureRecognizer(target: self,
                                                           action: #selector(HogeTableViewController.cellLongPressed(_ :)))

    // `UIGestureRecognizerDelegate`を設定するのをお忘れなく
    longPressRecognizer.delegate = self

    // tableViewにrecognizerを設定
    tableView.addGestureRecognizer(longPressRecognizer)
    
}
```
## extensionの再設定
これでセルが長押しされた際にコンテキストメニューが表示されるようになったはずです
しかし、このままでは表示するだけで何かを実行することはできないので、最初に追記した`extension`を再び編集します

```swift: HogeTableViewController.swift

// HogeTableViewController: ContextMenuDelegate 内の関数を上書きしています

/**
    コンテキストメニューの選択肢が選択された時に実行される
    - Parameters:
        - contextMenu: そのコンテキストメニューだと思われる
        - cell: **選択されたコンテキストメニューの**セル
        - targetedView: コンテキストメニューの発生源のビュー
        - item: 選択されたコンテキストのアイテム(タイトルとか画像とかが入ってる)
        - index: **選択されたコンテキストのアイテムの**座標
 */
func contextMenuDidSelect(_ contextMenu: ContextMenu,
                          cell: ContextMenuCell,
                          targetedView: UIView,
                          didSelect item: ContextMenuItem,
                          forRowAt index: Int) -> Bool {
	//長押しされたセルを、型を再指定して読み込むことができます
	let pressedTableCell = targetedView as! HogeTableViewCell
	
	//セル内の情報を取得したい場合
	let someImage: UIImage = pressedTableCell.someImageView.image
	let hugaTitle: String = pressedTableCell.hugaLabel.text
	//このようにして取得できます
	
	//選択されたコンテキストメニューのアイテム => item
	print("選択されたメニューのタイトル: ", item.title)
	
	
	//選択されたメニューのindexで条件分岐させたい場合
	switch index {
	case 0:
	    //0番目のセル(1番上のメニューがタップされると実行されます)
	    //この例では編集メニューに設定してあります
	    print("編集が押された!")
	
	case 1:
	    //同様です
	    print("削除が押された!")
	
	default:
	    //ここはその他のセルがタップされた際に実行されます
	    break
	}
	
	//最後にbool値を返します
	return true
	
}

// この下にも関数が３つあります

```

コンテキストメニューが選択されると`contextMenuDidSelect()`が呼ばれるので、この例を参考に改造していってください

# 最後に
個人で使ってみて使いやすかったのですが、日本語の紹介記事がなかったので、同じようなことをしている人の助けになればと思い、書かせていただきました
もし、間違っているところなどがあれば指摘してください