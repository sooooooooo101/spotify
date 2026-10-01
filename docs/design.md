# TBPN 英日字幕プレーヤー 設計書（ドラフト v0.1）

## 1. 目的

TBPN（Technology Business Programming Network）のアーカイブ回を、英語と日本語の2段字幕付きで聴けるようにする。英語学習とテック業界のキャッチアップが目的。iPhone では他のアプリを使いながら PiP（小窓）で字幕を見られるようにする。

### やらないこと（v1）

- Spotify アプリとの連携（Spotify の公開 API では音声ストリームを取得できず、PiP も Spotify 側の仕様に依存するため）
- ライブ配信中のリアルタイム翻訳
- 字幕の公開・配布（個人の学習用途に限る）

## 2. 前提と確認状況

| 項目 | 状況 |
|---|---|
| TBPN の公開 RSS（https://feeds.transistor.fm/technology-brother） | 存在は確認済み。中身（音声のみか動画か、公式の文字起こしが付いているか）は未確認 |
| 利用端末 | Mac と iPhone |
| Apple Developer Program | 加入済み |
| 翻訳 | Apple Translation フレームワークで開始し、後から Claude API に差し替えられる構造にする |
| 文字起こし | Mac で処理し、iCloud 経由で iPhone に同期 |

## 3. 全体構成

```
[TBPN RSS] ──> Mac アプリ
                 ├─ エピソード取得・音声ダウンロード
                 ├─ 文字起こし（英語・タイムスタンプ付き）
                 ├─ 文分割 → 翻訳（日本語）
                 ├─ 字幕 JSON を iCloud Drive に保存
                 └─ プレーヤー（Mac でも視聴可）
                         │ iCloud Drive
                         ▼
               iPhone アプリ
                 ├─ 音声は RSS から直接ストリーミングまたはダウンロード
                 ├─ 字幕 JSON を読み込み
                 └─ プレーヤー＋字幕 PiP
```

- SwiftUI のマルチプラットフォームアプリ（iOS / macOS）を1つ作る。
- 共通部分は Swift Package `TBPNCore` に切り出す（モデル、RSS パーサ、字幕ストア、プレーヤー、PiP 描画）。
- 文字起こしと翻訳のパイプラインは macOS ターゲットだけで有効にする。

## 4. データモデル

```swift
struct Episode: Codable, Identifiable {
    let id: String          // RSS の guid
    let title: String
    let publishedAt: Date
    let audioURL: URL
    let duration: TimeInterval?
}

struct Segment: Codable {
    let start: TimeInterval
    let end: TimeInterval
    let en: String
    var ja: String?         // 翻訳前は nil
}

struct Transcript: Codable {
    let episodeID: String
    let engine: String      // 例: "speechanalyzer" / "whisperkit-large-v3"
    let translator: String  // 例: "apple-translation" / "claude"
    let segments: [Segment]
    let schemaVersion: Int
}
```

保存先は iCloud Drive の ubiquity container 内にある `Transcripts/<episodeID>.json`。CloudKit より単純で、中身を目で確認しやすいためこちらを選ぶ。

## 5. 処理パイプライン（Mac）

1. 取得：RSS を定期的に確認し、新しいエピソードの音声をダウンロードする（3時間で数百 MB を想定）。
2. 文字起こし：`Transcriber` プロトコルの裏に実装を差し替えられるようにする。
   - 候補A：Apple SpeechAnalyzer / SpeechTranscriber（macOS 26 以降・端末内）
   - 候補B：WhisperKit（Core ML 版 Whisper）
   - どちらを採用するかは Phase 0 で、精度（固有名詞・早口・複数話者）と処理時間を実測して決める。
3. 文分割：単語タイムスタンプを文単位にまとめる。1セグメントは最大2行程度・約7秒を目安にする。
4. 翻訳：`Translator` プロトコルの裏に実装を置く。
   - v1：Apple Translation フレームワーク（en→ja）
   - 将来：Claude API。前後の文脈を渡して口語・スラング・業界用語の訳を改善する。
   - 文単位ではなく前後数文をまとめて送る設計にしておく（Claude に切り替えたとき品質に効くため）。
5. 保存：途中結果も保存し、中断しても再開できるようにする。

## 6. 再生と字幕表示

- AVPlayer で再生する。バックグラウンド再生（Audio background mode）、ロック画面の操作（MPNowPlayingInfoCenter）、再生速度の変更（0.75〜1.25倍）に対応する。
- アプリ内では英日2段の字幕を表示し、現在のセグメントを強調する。字幕をタップするとその位置にシークする。
- PiP：
  - `AVPictureInPictureController.ContentSource(sampleBufferDisplayLayer:playbackDelegate:)` を使う（iOS 15 以降・macOS 12 以降）。
  - 字幕を CoreGraphics で画像に描画し、CMSampleBuffer として AVSampleBufferDisplayLayer に送る。描画するのはセグメントが切り替わったときだけにする（低 fps）。
  - `playbackDelegate` で再生・一時停止とシークを AVPlayer に橋渡しする。

## 7. 段階計画

### Phase 0：技術検証（ここで止まる可能性がある項目を先に潰す）

- [ ] S1：iPhone 実機で、音声のみ再生しながらカスタム描画した字幕 PiP が出るか。バックグラウンドでも更新され続けるか。
- [ ] S2：3時間の回を Mac で文字起こしする。SpeechAnalyzer と WhisperKit について、処理時間と精度を比較する。
- [ ] S3：Apple 翻訳の品質を、TBPN の実際の文字起こし 10 分ぶんで評価する。
- [ ] S4：RSS の中身（音声・動画・公式文字起こしの有無）を確認する。

### Phase 1：Mac MVP

RSS 一覧 → 1エピソードを処理 → 英日字幕付きで再生 → Mac で PiP。

### Phase 2：iPhone 対応

iCloud 同期、iPhone プレーヤー、字幕 PiP。

### Phase 3：学習機能（任意）

単語をタップして解説、単語帳、話題ごとの日本語要約、Claude 翻訳への切り替え。

## 8. リスクと対策

| リスク | 対策 |
|---|---|
| カスタム字幕 PiP がバックグラウンドで止まる、または審査で問題になる | 個人利用なら TestFlight や開発用ビルドで使えれば足りる。S1 で最初に検証する |
| 3時間の文字起こしが遅い | Mac の常駐機能（ログイン項目）で夜間に処理する |
| 固有名詞の誤認識（社名・人名） | 用語辞書を用意し、Claude に切り替えた後は文脈付きで補正する |
| Apple 翻訳で口語が不自然になる | `Translator` を差し替え可能にしておく |
| 字幕のタイミングずれ | セグメント単位で手動オフセット調整を UI に用意する |
| 権利 | 字幕は端末と個人の iCloud 内に留め、公開・共有機能は作らない |

## 9. 開発の進め方に関する注意

- iOS / macOS のビルドと実機確認には Xcode が必要。Linux のクラウド環境ではビルドもテストもできないので、実装は Mac 上の Claude Code で行う。
- `TBPNCore` の純粋なロジック（RSS パース、文分割、字幕の時刻検索）はプラットフォームに依存しないように書き、`swift test` で検証できるようにする。
- 設計の正本はこのファイルとする。ChatGPT での設計変更、Antigravity での監査の結果はここに反映する。

## 10. 未決事項

- 文字起こしエンジン（S2 の結果で決める）
- 音声を iPhone でストリーミングするかダウンロードするか
- 最低対応 OS（SpeechAnalyzer を使うなら macOS 26 以降）
