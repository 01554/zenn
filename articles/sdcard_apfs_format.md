---
title: "SDカードをAPFSフォーマットするには"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [mac,jetdrive,apfs]
published: true
---


お疲れ様です波浪です。

愛機MacBookPro14のSSDが枯渇したのでJetDriveLite330を買ったんですが

https://jp.transcend-info.com/product/memory-card/jetdrive-lite-330

こいつ、MacBook専用設計の癖に出荷時はEx-FATでフォーマットされています。

これのせいで OrbStackのストレージフォルダをJetDriveLiteに設定しようとすると
`validate data dir: data storage location must be formatted as APFS`
とおこられてしまいます。

んじゃ、フォーマットすりゃいいんじゃろ、と普段通りディスクユーティリティを起動するんですが
ここからAPFSフォーマットにすることはできません！！！

![](/images/sdcard_apfs_format/diskutil.png)


んじゃどうするんだよ？ってなりますね。
はい、解決策は
`diskutil eraseDisk APFS "好きな名前" SDカードの位置`
です。


まず `diskutil list`でSDカードを特定します、自分の場合は `/dev/disk4`ですね

位置が特定できたら
`diskutil eraseDisk APFS "ExternalStorage" /dev/disk4`

![](/images/sdcard_apfs_format/cli.png)



ご覧の有様ですね

ディスクユーティリティ上でもAPFSになっていることが確認できます

![](/images/sdcard_apfs_format/apfs.png)


以上、よろしくお願いいたします。