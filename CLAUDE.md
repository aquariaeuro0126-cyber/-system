# 備品管理システム

## プロジェクト概要
備品の管理を行うWebアプリケーション。GitHub Pagesで公開中。

## 現在の状態
- GitHub Pages公開中：リポジトリルートのindex.html
- GitHubリポジトリ：git@github.com:aquariaeuro0126-cyber/-system.git
- ローカル：v1.0.0〜v1.3.0のフォルダで手動管理していた
- git管理：2026年5月より開始

## 今後の方針
- 新しい変更はindex.htmlを直接編集してコミットする
- バージョンフォルダは新たに作らない（gitが履歴を管理するため）
- 過去のv1.0.0〜v1.3.0フォルダはそのまま残す（削除しない）
- GitHub PagesはリポジトリルートのURLで公開中

## ファイル構成
```
備品管理システム/
├── index.html   ← 公開中の最新版（直接編集する）
├── v1.0.0/      ← 過去バージョン（残すだけ・編集しない）
├── v1.1.0/
├── v1.2.0/
└── v1.3.0/      ← git管理開始前の最終手動バージョン
```

## 作業時の注意
- index.htmlを編集したら必ずコミット＆pushする
- pushの前に変更内容をユーザーに報告・承認を得ること

## 作業記録

### 2026-06-02（Firebase Firestore連携の追加とバグ修正）

#### 背景と出発点
v1.3.0までLocalStorageのみで動作していた備品管理システムに、Firebase Firestoreを使ったクラウド保存・複数端末間同期機能を追加した（v1.4.0）。また、前セッションから引き継いでいたv1.3.0のspec.txt更新も実施した。

なお、本セッションから方針変更：バージョンフォルダを作らず、`index.html`を直接編集してGitが履歴を管理する運用に切り替えた。

#### Firebase SDK方式の選定

**動的import()方式の失敗**：
最初はmodular SDK（v12）を`import()`で動的読み込みする方式を試みたが、iOS Safariや一部環境でCORSエラーが発生した。

**compat SDK（v9.22.2）への切り替え**：
`<script src="https://www.gstatic.com/firebasejs/9.22.2/firebase-app-compat.js">`方式に変更。これによりSDKが確実に読み込まれ、`firebase.initializeApp()`でグローバルに初期化できる。

重要な差異：modular SDKでは`snap.exists()`（メソッド）、compat SDKでは`snap.exists`（プロパティ）であるため、変更時に修正が必要だった。

#### 同期戦略の設計（3回の変遷）

**第1版（シンプル版）**：Firestoreにデータがあれば読み込む、なければローカルを移行。実装後、「更新ボタンを押すとデータが元に戻る」問題が発生。

**第2版（ローカル優先）**：`hasLocalData`フラグを導入し、ローカルデータがある端末ではローカルを正とする方式に変更。しかしこれが新たな問題を引き起こした。

**第3版（Firestore優先）最終採用**：
```javascript
const itemsSnap = await _fsDb.collection('data').doc('items').get();
if (itemsSnap.exists) {
  await syncFromFirestore();   // Firestoreが正
} else {
  await migrateLocalToFirestore();  // 初回セットアップのみ
}
```
第2版の問題：Device Bはアプリ起動時に初期シードデータ（eq_items）がLocalStorageに入るため`hasLocalData=true`になり、`migrateLocalToFirestore()`が呼ばれてFirestoreを初期データで上書きしていた。Firestoreの`data`コレクションはユーザーがコンソールで存在を確認済みだったため、書き込み自体は機能しており、同期失敗ではなく「上書き」が問題だった。

#### 管理画面で保存ボタンが押せない問題の修正

**症状**：管理画面を開いた状態で、Firebase初期化IIFE完了後に`renderItems()`が呼ばれ、DOM干渉が起きていた。

**修正**：
```javascript
if (!document.getElementById('screen-admin').classList.contains('active')) {
  renderItems();
  checkOverdue();
}
```
管理画面表示中はrenderItems()を呼ばないガード条件を追加。

#### Firestore同期エラー（invalid-argument）の診断と原因特定

**症状**：管理画面で編集・保存すると⚠️アイコンが表示され同期が失敗する。

**診断方法**：`syncToFirestore`のcatchブロックにエラーコードをトースト表示する機能を追加してpushし、ユーザーに再現してもらった。

**結果**：`invalid-argument`（データサイズ超過）エラーを確認。

**原因**：Firestoreのドキュメントサイズ上限は1MiB（約1MB）。備品画像（Base64）やカスタム音声（Base64）がそのままFirestoreドキュメントに含まれるとサイズを超過する。スマホ撮影の写真は1枚で3〜5MB相当のBase64になる。

#### Firebase Storageに関する方針の訂正

当初「Firebase Storageは有料プランが必要」と説明していたが、実際にはSpark（無料）プランで5GBまで利用可能であることが判明。

**ユーザーの希望**：画像・音声も含めて他端末と共有したい。

**解決策の比較**：
- プランA（画像圧縮）：canvas APIで400×400px・JPEG80%に圧縮→20〜50KB、Firestoreに保存可。追加設定不要。
- プランB（Firebase Storage）：画像/音声をStorageに保存、URLをFirestoreに保持。元画質のまま。SDK追加・セキュリティルール設定が必要。

**ユーザーの選択**：プランB（Firebase Storage）に挑戦する。

#### 決定事項・次のタスク
- v1.4.0として`index.html`直接編集方式に移行済み（バージョンフォルダは作らない）
- コミット履歴：`4b1bd8a`→`ccfd2f1`→`50944f4`→`ec17cb2`→`d6ab2fe`
- Firebase Storageの無料枠（5GB）を使って画像・音声を同期する実装が次のタスク
  - Firebase Storage SDKの追加
  - 画像アップロード処理の変更（Base64→StorageURL）
  - セキュリティルールの設定（Firestoreと同様のテストモード）
  - 既存のBase64画像データの移行処理

### 2026-06-02（第2セッション：Firebase Storage連携の追加）

#### 背景と出発点
前セッションでFirestore連携（v1.4.0）を実装したが、管理画面での編集時に「データサイズ超過（invalid-argument）」の同期エラーが発生した。原因究明のためエラーコードをトーストで表示する診断コードを追加し、原因がFirestoreの1MiB制限であることを特定した。

根本的な問題は「画像（Base64）・カスタム音声（Base64）をそのままFirestoreドキュメントに保存しようとしていたこと」。ユーザーの当初の動機が「備品画像を複数端末で共有したい」だったため、単純にBase64を除外するだけでは要件を満たせない。

#### 解決策の選定（プランA vs プランB）

**プランA（画像圧縮）**：canvas APIで400×400px・JPEG80%に圧縮してFirestoreに保存。追加設定不要・実装が小さい。

**プランB（Firebase Storage）**：画像・音声をStorageに保存しURLをFirestoreに持つ。元画質のまま同期可能。SDK追加・セキュリティルール設定が必要。

**選択：プランB**。ユーザーの要望が「元の画像を他端末でも見たい」だったため。

なお、本セッションで「Firebase Storageは有料プランが必要」という前セッションの誤った説明を訂正した。実際にはSpark（無料）プランで5GBまで利用可能。

#### Firebase Storageのセットアップ（ユーザー作業）

Firebase Consoleで「Storage」→「始める」→テストモード→ロケーション選択の手順を案内。「料金不要のロケーション」の選択肢がUS東・西・中部の3つのみ（アジアは有料プランのみ）だったため、`us-central1`を選択。テストモードのセキュリティルールを手動で有効期限まで設定して完了。

#### Firebase Storage実装（コード変更17箇所）

**設計の考え方**：

データ構造を以下のように分離した：
- `image`（Base64）：LocalStorageのみ保持・高速なローカル表示用
- `imageUrl`（Storage URL）：Firestoreに保存・他端末との同期用
- `customSound`（Base64）：LocalStorageのみ保持・オフライン再生用
- `customSoundUrl`（Storage URL）：Firestoreに保存・他端末との同期用

Firestoreへの書き込み時はBase64フィールドを除外し、読み込み時はFirestoreデータとローカルBase64をマージする設計にした。これにより：
- アップロードした端末：Base64（高速）とURLの両方で表示・再生可能
- 他の端末：URLで画像・音声を取得（Storage経由）

**実装した変更の概要**：

1. Firebase Storage SDK（`firebase-storage-compat.js`）の追加
2. `uploadToStorage(path, base64)`関数：Base64→Blob変換→Storage putでアップロードしてURLを返す
3. `syncToFirestore()`：itemsのBase64 `image`、settingsの`customSound`を除外してFirestoreに書き込む
4. `syncFromFirestore()`：読み込み時にローカルBase64と`imageUrl`/`customSoundUrl`をマージ
5. `saveItem()`：async化。新規画像があればStorageにアップロードして`imageUrl`に格納。`existingImageUrl`変数を追加し編集時の既存URLを保持
6. `handleSoundUpload()`：async化。Storageに音声アップロードして`customSoundUrl`を保存
7. `renderItems()`/`renderAdminItems()`/`editItem()`：`imageUrl || image`で表示
8. `playCustom()`/`renderSoundOptions()`/`updateCustomSoundLabel()`：`customSoundUrl`にも対応
9. `clearCustomSound()`：`customSoundUrl`も同時に削除

**`existingImageUrl`変数の追加理由**：
編集モードで既存画像を持つアイテムを開いた時、ユーザーが新しい画像を選ばずに保存した場合、既存の`imageUrl`を消さないよう保持する必要があった。`uploadedImage`（Base64）とは別に管理する変数として追加。

#### 決定事項・次のタスク
- v1.5.0としてコミット`46cb586`をGitHubにpush済み
- Firebase StorageのセキュリティルールはFirestoreと同様のテストモード（有効期限付き）
- 次のタスク：動作確認（翌日予定）
  - 管理画面で画像付き備品を保存 → ☁️が表示されるか確認
  - 別端末でアクセス → 同じ画像が表示されるか確認
  - カスタム音声のアップロード → 別端末で再生できるか確認

### 2026-06-02（第3セッション：同期が効かない問題の徹底的な原因究明）

#### 背景と出発点
第2セッションでFirebase Storage連携（v1.5.0）を実装し、動作確認は翌日（実際は同日中）に行うことになっていた。しかし新しい環境で確認したところ、**Storage実装後も依然として「変更が記録されない・他端末で更新されない・更新ボタンを押すと元に戻る」という問題が再発**していた。本セッションは、表面的な対症療法ではなく真の根本原因に到達するまで診断を重ねた記録である。最終的に3つの独立した原因を発見・解決した。

#### 問題① クロスデバイス同期の戦略誤り（再確認・修正済みだったが不十分だった）

セッション冒頭、まず前回までの「ローカル優先」同期戦略を「Firestore優先」に修正した。
- 旧：`hasLocalData`があればローカルを正としてFirestoreに移行 → 新端末の初期シードがFirestoreを上書き
- 新：`itemsSnap.exists`ならFirestoreを正として読み込み、空のときのみローカルを移行

ユーザーがFirestoreコンソールで`data`コレクションの存在を確認したため、書き込み自体は機能していると判断できた。しかしこの修正だけでは問題は解決せず、より深い原因が隠れていた。

#### 問題② Firestore同期エラーの可視化と「データサイズ超過」の特定

「管理画面で編集すると同期エラーが起きる」という報告を受け、原因を特定するため`syncToFirestore`のcatchブロックにエラーコードを日本語トーストで表示する診断コードを追加した。

**結果**：`invalid-argument`（データサイズ超過）を確認。Firestoreの1ドキュメント上限は1MiB。画像Base64がそのままFirestoreに入るとサイズ超過する。これは第2セッションのStorage導入（画像をStorageに分離）で解決した。

#### 問題③【最重要】診断は全て正常なのに同期されない矛盾 → 初期データ競合の発見

Storage実装後も問題が続いたため、**一度の操作で全てを切り分けられる包括的な診断パネル**を管理画面の設定タブに新設した（`runSyncDiagnosis()`）。SDK読み込み・Firestore初期化・Storage初期化・Firestore読み込み・Firestore書き込み・Storage書き込みを順にテストし、各段階の成否とエラーコードを画面表示する設計。

**診断結果**：全項目が✅（書き込み・読み込みとも正常）。しかし変更は反映されない。この「正常なのに動かない」矛盾が決定的なヒントとなった。

**注目点**：診断の「items 読み込みOK（**10件**）」。この10件は初期登録備品の数と完全に一致。つまりFirestoreにはユーザーの編集ではなく**初期データが入っていた**。

**原因究明の過程**：
`seedInitialItems()`（初期10件投入）の実行タイミングを調査。この関数は`DOMContentLoaded`で`init()`から無条件に呼ばれ、localStorageが空なら`DB.saveItems(initialItems)`でFirestoreに初期データを書き込んでいた。一方、Firebase初期化IIFEは即時実行で非同期にFirestoreを読む。両者が競合し、以下のシーケンスが発生していた：

```
① 端末Aで編集 → Firestoreに編集データ保存 ✅
② 別端末でアプリを開く（localStorageが空）
③ DOMContentLoaded → seedInitialItems() が初期10件をFirestoreに書き込み → 上書き ❌
④ 端末Aで更新 → Firestoreの初期データを読む → 元に戻る
```

**修正**：初期データ投入をFirebase初期化IIFEに統合し、**Firestoreが空のときだけ**実行するよう変更。
- `init()`から`seedInitialItems()`を削除
- IIFE内：`itemsSnap.exists`ならseedせず読み込みのみ、空のときだけseed＆移行
- `firebaseOk`フラグを追加し、オフライン時のみローカルでseed

この競合こそが「編集が反映されない／更新で元に戻る」ループの真因だった。診断が全て正常だったのは、診断が`_diagnosis`ドキュメントへの書き込みをテストしていて、`items`ドキュメントへの上書き競合は検出できなかったため。

#### 問題④ 保存ボタンの二重クリックによる挙動破綻

ユーザーから「保存ボタンを押してから完了まで時間がかかり、その間に再クリックして挙動が狂う」との指摘。`saveItem()`はStorageアップロード（ネットワーク待ち）でasync化されており、完了前に再クリック可能だった。

**修正**：
- 保存ボタンに`id="btn-save-item"`を付与
- `_savingItem`フラグで二重実行をブロック
- 処理中はボタンを無効化し「⏳ 保存中...」表示、`finally`で必ず復元
- CSSに`.btn-primary:disabled`（グレー・半透明）を追加

#### 問題⑤【根本問題】画像3枚目で保存できない → localStorage容量超過

「2枚は登録できたが3枚目で保存できず、キャンセルボタンが保存ボタンに隠れて押せない。保存もされない」という報告。

**原因究明**：症状（特定枚数で失敗・レイアウト崩れ・保存されない）から、`localStorage.setItem`の`QuotaExceededError`（容量超過）と判断。画像をStorageに上げて`imageUrl`で管理しているにもかかわらず、`DB.saveItems`がBase64の`image`フィールドもlocalStorage（上限約5MB）に二重保存していた。写真は1枚2〜4MB相当のBase64になるため3枚目で超過。例外で`saveItems`が中断し、後続のフォームリセット・ボタン位置復元が実行されずレイアウトが崩れていた。

**修正**：
- `saveItems`：localStorage保存時に`image`（Base64）を除外。`try-catch`で容量超過時はトースト表示
- `saveSettings`：同様に`customSound`（Base64）を除外
- `syncFromFirestore`：ローカルBase64を復元するマージを廃止し、Firestoreの内容（URLのみ）をそのまま保存。これにより**起動時に既存の溜まったBase64も一掃**される
- 表示・再生は全て`imageUrl`/`customSoundUrl`（Storage URL）に統一

これでlocalStorageにはBase64が一切入らなくなり、画像は何枚でも登録可能になった。

#### 決定事項・次のタスク
- 3つの独立した原因（初期データ競合・二重クリック・localStorage容量超過）を全て解決し、満足のいく挙動を確認
- コミット履歴：`ec17cb2`→`d6ab2fe`→`8d43dc8`（診断機能）→`b5fc6a5`（初期データ競合修正）→`68f0e12`（二重クリック防止）→`2072b65`（容量超過修正）
- **重要な教訓**：
  - 「診断が全て正常なのに動かない」場合、診断がテストしていない経路（本セッションでは初期化時の書き込み競合）を疑うべき
  - 非同期の初期化処理（IIFE）と`DOMContentLoaded`の処理は実行順序が競合しうる。データ書き込みを伴う初期化は一箇所に集約する
  - クラウド（Storage）に保存したバイナリをlocalStorageに二重保存しない。localStorageは軽量データ専用とする
- 同期診断パネル（`runSyncDiagnosis()`）は今後のトラブル時にも有用なため常設とする
- 次のタスク：実運用での最終動作確認（画像複数枚・複数端末での共有）

### 2026-06-02（第4セッション：貸出フローに選択取り消しボタンを追加）

#### 背景と出発点
同期問題（第3セッション）が解決し満足のいく挙動になった後、ユーザーから貸出フローの利便性改善要望があった。現在の貸出は「備品をタップ → 即座に名前入力へ進む」フローのため、**間違えて別の備品をタップしてしまった場合に取り消す手段が分かりにくい**という課題。間違えた備品を再タップして解除する必要があり直感的でなかった。

#### 設計の考え方

**取り消し範囲の確認**：
実装前に「直前にタップした1つだけ取り消す」か「選択を全部リセットする」かをユーザーに確認した（仕様の根幹に関わるため）。ユーザーは**「選択をすべてリセット」**を選択。1つずつ借りる運用が主のため、全リセットの方がシンプルで分かりやすいという判断。

**ボタンの配置とデザイン**：
備品タップ後に表示される「他にも借りますか？」モーダル（`overlay-more`）内に3つ目のボタンとして配置。既存の「→ 名前を入力して貸出する」（btn-primary）・「＋ 他にも追加する」（btn-secondary）より控えめな赤文字のテキストボタンにした。誤って押しにくくしつつ、取り消し操作であることが色で識別できるようにする狙い。

#### 実装内容
- **HTML**：`overlay-more`モーダルに「✕ 選択をやめる（最初に戻る）」ボタンを追加
- **JS**：`cancelSelection()`関数を追加。`selectedItems = []`で選択をクリア → `renderItems()`で画面更新 → `closeMoreModal()`でモーダルを閉じる
- **CSS**：`.btn-cancel-select`クラスを追加（透明背景・赤文字・控えめなテキストボタン、`:active`で薄赤背景）

#### 決定事項・次のタスク
- コミット`5342208`（feat: 貸出フローに選択取り消しボタンを追加）をGitHubにpush済み
- 取り消しは「全選択リセット」方式を採用（ユーザー確認の結果）
- 将来的に「直前の1つだけ取り消す」方式の要望が出れば追加検討
- 次のタスク：実運用でのフィードバック対応（現時点では未定）
