---
title: "【Flutter】DropdownMenuFormField で Form バリデーションする"
emoji: "⚾"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "material", "form"]
published: true
---

# 1. はじめに

[DropdownMenu](https://api.flutter.dev/flutter/material/DropdownMenu-class.html) を Form と組み合わせて使いたい…そんなときに便利なのが [DropdownMenuFormField](https://main-api.flutter.dev/flutter/material/DropdownMenuFormField-class.html) です。

以下の PR でマージされました。ただし、現時点（2025-08-10）ではまだ stable に入っていません。（次のstableのマイナーバージョンアップで入るのでは？）

- PR: https://github.com/flutter/flutter/pull/163721
- API ドキュメント (main channel): https://main-api.flutter.dev/flutter/material/DropdownMenuFormField-class.html

# 2. サンプルコード（Form + バリデーション）

こちらのようなサンプル実装してみました。
https://dartpad.dev/?id=5fdd43d481bf223aeceeab9012da0d80&channel=main&run=true

必須選択の `DropdownMenuFormField` を 2 つ（果物と色）配置し、`validator` で未選択時にエラー表示、Submit 時に snackbar を出す例です。`GlobalKey<FormFieldState<T>>` を使い、選択時に `validate()` を走らせてその場でエラーを消しています。

![](https://storage.googleapis.com/zenn-user-upload/8e98231fbd69-20250810.gif =300x)

# 3. 注意点: Form の reset ではクリアされない

`DropdownMenu` は `FormState.reset()` だけではクリアされません。関連 Issue は以下です。

https://github.com/flutter/flutter/issues/165877

そのため、上のサンプルでは以下を合わせて行っています。

- `FormState.reset()` を呼ぶ
- 各 `FormFieldState<T>` に対して `reset()` を呼ぶ
- 各 `TextEditingController` を `clear()` する
- `setState` で保持している選択値を `null` に戻す

これで UI 上の表示と内部状態の両方をリセットできます。

# 4. おわりに

[DropdownMenu](https://api.flutter.dev/flutter/material/DropdownMenu-class.html) は、Material 3 を意識した後発の Widget ですね。
[DropdownMenuFormField](https://main-api.flutter.dev/flutter/material/DropdownMenuFormField-class.html) が用意されたことで、DropdownMenu と Form を組み合わせたいときにも対応できるようになりました。

今後 stable へ反映されれば、Material 3 対応フォームの実装がより一貫したものになりそうですね。
