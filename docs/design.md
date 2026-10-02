# TBPN 英日字幕プレーヤー 設計書（ドラフト v0.2）

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
    let id: String          // RSS の guid（外部入力。ファイル名には使わない）
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
    let episodeID: String   // RSS の guid
    let engine: String      // 例: "speechanalyzer" / "whisperkit-large-v3"
    let translator: String  // 例: "apple-translation" / "claude"
    let segments: [Segment]
    let schemaVersion: Int
}
```

保存先は iCloud Drive の ubiquity container 内にある `Transcripts/<fileKey>.json`。CloudKit より単純で、中身を目で確認しやすいためこちらを選ぶ。

- `fileKey` は `SHA256(guid)` の16進文字列にする。guid をそのままファイル名に使うと、`/` や `..` を含んだ場合に保存先の外へ書き込めてしまうため（SEC-001）。
- 読み込み時に JSON のデコードや `schemaVersion` の検証に失敗した場合は、クラッシュさせずに「要再処理」として扱う（SEC-006）。

## 5. 処理パイプライン（Mac）

1. 取得：RSS を定期的に確認し、新しいエピソードの音声をダウンロードする（3時間で数百 MB を想定）。URL の検証とサイズ上限は 9章を参照。
2. 文字起こし：`Transcriber` プロトコルの裏に実装を差し替えられるようにする。
   - 候補A：Apple SpeechAnalyzer / SpeechTranscriber（macOS 26 / iOS 26 以降・端末内）
   - 候補B：WhisperKit（Core ML 版 Whisper）
   - どちらを採用するかは Phase 0 で、精度（固有名詞・早口・複数話者）と処理時間を実測して決める。
3. 文分割：単語タイムスタンプを文単位にまとめる。1セグメントは最大2行程度・約7秒を目安にする。
4. 翻訳：`Translator` プロトコルの裏に実装を置く。
   - v1：Apple Translation フレームワーク（en→ja）
   - 将来：Claude API。前後の文脈を渡して口語・スラング・業界用語の訳を改善する。API キーとプロンプトの扱いは 9章を参照。
   - 文単位ではなく前後数文をまとめて送る設計にしておく（Claude に切り替えたとき品質に効くため）。
5. 保存：途中結果は Mac ローカルの Application Support に保存し、中断しても再開できるようにする。iCloud の `Transcripts/` に置くのは完成版だけにし、`NSFileCoordinator` を使ってアトミックに書き込む。こうすれば、書きかけのファイルが iPhone に同期されることはない（SEC-006）。

## 6. 再生と字幕表示

- AVPlayer で再生する。バックグラウンド再生（Audio background mode）、ロック画面の操作（MPNowPlayingInfoCenter）、再生速度の変更（0.75〜1.25倍）に対応する。
- アプリ内では英日2段の字幕を表示し、現在のセグメントを強調する。字幕をタップするとその位置にシークする。字幕は `Text(verbatim:)` で表示し、Markdown として解釈させない（SEC-007）。
- PiP：
  - `AVPictureInPictureController.ContentSource(sampleBufferDisplayLayer:playbackDelegate:)` を使う（iOS 15 以降・macOS 12 以降）。
  - 字幕を CoreGraphics で画像に描画し、CMSampleBuffer として AVSampleBufferDisplayLayer に送る。描画するのはセグメントが切り替わったときだけにする（低 fps）。描画の前に制御文字を除去し、表示枠に収まらない部分は省略記号で切り詰める（SEC-007）。
  - `playbackDelegate` で再生・一時停止とシークを AVPlayer に橋渡しする。

## 7. 段階計画

### Phase 0：技術検証（ここで止まる可能性がある項目を先に潰す）

- [ ] S1：iPhone 実機で、音声のみ再生しながらカスタム描画した字幕 PiP が出るか。バックグラウンドでも更新され続けるか。
- [ ] S2：3時間の回を Mac で文字起こしする。SpeechAnalyzer と WhisperKit について、処理時間と精度を比較する。
- [ ] S3：Apple 翻訳の品質を、TBPN の実際の文字起こし 10 分ぶんで評価する。
- [ ] S4：RSS の中身を確認する（音声か動画か、公式の文字起こしがあるか、enclosure が HTTPS か、ファイルサイズの実測値）。サイズの実測値をもとに、9章のダウンロード上限を確定する。

### Phase 1：Mac MVP

RSS 一覧 → 1エピソードを処理 → 英日字幕付きで再生 → Mac で PiP。

受入基準（セキュリティ）：guid のハッシュ化、URL の検証、サイズ上限と空き容量チェックについて、`TBPNCore` の単体テストがパスしていること（S-1〜S-3）。

### Phase 2：iPhone 対応

iCloud 同期、iPhone プレーヤー、字幕 PiP。

受入基準（セキュリティ）：破損した JSON や書きかけの JSON を読み込んでもクラッシュしないこと（S-4）。

### Phase 3：学習機能（任意）

単語をタップして解説、単語帳、話題ごとの日本語要約、Claude 翻訳への切り替え。

受入基準（セキュリティ）：API キーが Keychain 以外に保存されていないこと。指示文を混ぜたセグメントを翻訳させても、出力が JSON スキーマの範囲に収まること（S-5、S-6）。

## 8. リスクと対策

| リスク | 対策 |
|---|---|
| カスタム字幕 PiP がバックグラウンドで止まる、または審査で問題になる | 個人利用なら TestFlight や開発用ビルドで使えれば足りる。S1 で最初に検証する |
| 3時間の文字起こしが遅い | Mac の常駐機能（ログイン項目）で夜間に処理する |
| 固有名詞の誤認識（社名・人名） | 用語辞書を用意し、Claude に切り替えた後は文脈付きで補正する |
| Apple 翻訳で口語が不自然になる | `Translator` を差し替え可能にしておく |
| 字幕のタイミングずれ | セグメント単位で手動オフセット調整を UI に用意する |
| 権利 | 字幕は端末と個人の iCloud 内に留め、公開・共有機能は作らない |

## 9. セキュリティ要件

前提とする脅威モデル：個人用のアプリで、入力元は固定した1つの RSS（Transistor の HTTPS 配信）だけ。サーバーは持たない。外部入力（RSS・音声・文字起こしテキスト）は信頼しないデータとして扱うが、対策は個人アプリの規模に見合う範囲にとどめる。

| ID | 要件 | 監査指摘 |
|---|---|---|
| S-1 | guid はファイル名に使わない。ファイル名には `SHA256(guid)` を使う | SEC-001 |
| S-2 | フィード URL はアプリに固定する。enclosure URL は `https` だけを許可し、ATS の例外は設定しない。音声として正しいかの確認は Content-Type ではなく、AVAsset で読み込めるかで判定する（CDN が `application/octet-stream` を返すことがあるため） | SEC-004 |
| S-3 | ダウンロードには上限を設ける。初期値は 2GB で、S4 の実測値をもとに見直す。`Content-Length` が上限を超えるか、受信中に上限を超えたら中断する。開始前に空き容量を確認し、5GB 未満なら処理しない。文字起こしは `thermalState` が `.serious` 以上、または低電力モードのときは延期する | SEC-005 |
| S-4 | iCloud への書き込みは完成版だけにし、`NSFileCoordinator` を使ってアトミックに行う。読み込みに失敗したら「要再処理」状態にする | SEC-006 |
| S-5 | Claude API キーは Keychain（`kSecClassGenericPassword`、`kSecAttrAccessibleWhenUnlockedThisDeviceOnly`）にだけ保存する。UserDefaults・plist・iCloud・ソースコードへの保存は禁止する。入力は `SecureField` で受け付ける。翻訳は Mac だけで行うので、キーを登録するのも Mac だけでよい | SEC-002 |
| S-6 | Claude に送る文字起こしは `<transcript>` タグで囲み、その中身は指示ではなくデータだとシステムプロンプトに明記する。出力は Structured Outputs で `[{ "id": Int, "ja": String }]` に固定し、送ったセグメントの id と一致しないものは捨てる。ツールは渡さない | SEC-003 |
| S-7 | 字幕は `Text(verbatim:)` で表示する。PiP に描画する前に制御文字を除去し、長すぎる文字列は切り詰める | SEC-007 |

採用しなかったもの：
- プライベート IP やループバックアドレスへの接続拒否（SEC-004 の一部）。フィード URL を固定し、HTTPS だけを許可すれば、残るリスクは Transistor 自体が乗っ取られる場合に限られる。その場合も、端末から内部ネットワークに届く経路は DNS 側で回避できてしまうため、IP 判定を入れてもほとんど効かない。
- 「macOS 26 は誤記なので 15 に直す」という指摘（SEC-008）。Apple は 2025 年から OS のバージョン番号を年号ベースに変えており、macOS 26（Tahoe）と iOS 26 は実在する。SpeechAnalyzer もこのバージョンで追加された API なので、表記はそのままにする。

## 10. 開発の進め方に関する注意

- iOS / macOS のビルドと実機確認には Xcode が必要。Linux のクラウド環境ではビルドもテストもできないので、実装は Mac 上の Claude Code で行う。
- `TBPNCore` の純粋なロジック（RSS パース、文分割、字幕の時刻検索）はプラットフォームに依存しないように書き、`swift test` で検証できるようにする。
- 設計の正本はこのファイルとする。ChatGPT での設計変更、Antigravity での監査の結果はここに反映する。

## 11. 未決事項

- 文字起こしエンジン（S2 の結果で決める）
- 音声を iPhone でストリーミングするかダウンロードするか
- 最低対応 OS（SpeechAnalyzer を使うなら macOS 26 / iOS 26 以降）
- ダウンロード上限の確定値（S4 の結果で決める）
