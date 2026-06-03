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
