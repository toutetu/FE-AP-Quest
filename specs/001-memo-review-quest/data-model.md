# Data Model: 第1弾 メモ・復習・クエスト

保存するデータは3種類（プロフィール、メモ、勉強の記録）だけで、それ以外の値はすべて計算で求める
（research.md R5）。日付はすべて「学習日」（午前4時区切り、`YYYY-MM-DD`）で表す。

## 保存場所

| データ | db のパス | 件数の目安 |
|---|---|---|
| プロフィール | `data/users/<uid>/profile` | 1件 |
| メモ | `data/users/<uid>/profile/memos/<memoId>` | 12月までに1,000〜3,000件 |
| 勉強の記録 | `data/users/<uid>/profile/studyLogs/<logId>` | 1日数件、12月までに数百件 |

`<uid>` は `user.id()` の値。`data/users/<uid>/` 以下は本人だけが読み書きできる。
id が取れないとき（サインインしていないなど）は保存の機能を止め、その旨を表示する。

## 保存するデータ

### Profile（プロフィール）

| 項目 | 型 | 初期値 | 決まりごと |
|---|---|---|---|
| `quests.dailyNewMemos` | 整数 | 5 | 0〜50 |
| `quests.dailyKakomon` | 整数 | 20 | 0〜200 |
| `quests.weeklyMinutes` | 整数 | 1440（24時間） | 0〜6000 |
| `jobKeywords` | 文字列の配列 | `["面接", "面談", "選考", "説明会"]` | 1語20文字以内、最大20語 |
| `jobDayOverrides` | `{ [学習日]: 真偽 }` | `{}` | 14日より古いものは保存時に削除 |
| `calendarId` | 文字列 または null | null | `list_calendars` で選んだ id。リポジトリには書かない |
| `freeTimeWindow` | `{ start: "HH:MM", end: "HH:MM" }` | `08:00`〜`23:00` | start < end |
| `bosses` | Boss の配列 | 下記の3体 | id は固定 |
| `timer` | `{ startedAt: 日時 }` または null | null | 勉強タイマーの開始時刻。端末をまたいで続けられる |
| `soundOn` | 真偽 | true | |
| `schemaVersion` | 整数 | 1 | 形を変えるときに上げる |

**Boss（ボス）**: `{ id, name, examDay, result }`

| id | name | examDay | result |
|---|---|---|---|
| `ap-a` | AP 科目A | 学習日 または null | `pending` / `passed` / `failed` / `unknown` |
| `ap-b` | AP 科目B | 同上 | 同上 |
| `fe` | FE | 同上 | 同上 |

### Memo（メモ・用語カード）

| 項目 | 型 | 必須 | 決まりごと |
|---|---|---|---|
| `id` | 文字列 | ○ | 端末側で作る。再送しても同じ id |
| `term` | 文字列 | ○ | 前後の空白を除いて1〜60文字 |
| `termKey` | 文字列 | ○ | 重複判定用。NFKC正規化・小文字化・空白除去した `term` |
| `explanation` | 文字列 | ○ | 1〜500文字 |
| `source` | Source または null | | 出会った問題 |
| `fieldMinor` | 1〜23 の整数 または null | | 中分類の番号（R13）。大分類は番号から求める |
| `createdAt` | 日時（ISO 8601） | ○ | |
| `createdDay` | 学習日 | ○ | |
| `updatedAt` | 日時 | ○ | |
| `srs.step` | 0〜5 の整数 | ○ | 間隔の段階。新規は0 |
| `srs.dueDay` | 学習日 | ○ | 次の出題日。新規は `createdDay` の翌日 |
| `srs.history` | Answer の配列 | ○ | 回答の履歴。最大200件（超えたら古い順に削除） |
| `ai` | AiResult または null | | AIチェックの結果 |

**Source（出会った問題）**: `{ exam: "AP" | "FE", year: string, season: "春" | "秋" | "特別" | "公開" | "", number: 整数 または null }`
例: R5秋 問23 → `{ exam: "AP", year: "R5", season: "秋", number: 23 }`

**Answer（回答）**: `{ day: 学習日, grade: "remembered" | "unsure" | "forgot", via: "card" | "quiz" }`

**AiResult（AIチェックの結果）**: `{ checkedAt, verdict, feedback, suggestedExplanation, pitfalls, related, suggestedFieldMinor, quiz, aiGenerated: true, editedByUser }`
各項目の形と検査は [contracts/ai-check.md](./contracts/ai-check.md) に従う。

### StudyLog（勉強の記録）

| 項目 | 型 | 必須 | 決まりごと |
|---|---|---|---|
| `id` | 文字列 | ○ | 端末側で作る |
| `day` | 学習日 | ○ | |
| `minutes` | 整数 | ○ | 0〜720 |
| `kakomonSolved` | 整数 | ○ | 0〜500 |
| `kakomonCorrect` | 整数 | ○ | 0〜`kakomonSolved` |
| `fieldMajor` | 1〜9 の整数 または null | | 分野ごとに記録するとき |
| `source` | `"timer"` / `"manual"` | ○ | |
| `createdAt` | 日時 | ○ | |

## 状態の移り変わり

### メモの段階（間隔反復, R9）

```text
段階:   0      1      2      3       4       5
間隔:   1日    3日    7日    14日    30日    60日（討伐済み）

覚えてた: 段階 s → min(s+1, 5)、出題日 = 今日 + 間隔[新しい段階]
あやしい: 段階 s → s、        出題日 = 今日 + 間隔[s]
忘れた:   段階 s → 0、        出題日 = 今日 + 1日
```

表示上の状態: 段階0で一度も回答していない →「新規」、段階0〜4 →「復習中」、段階5 →「討伐済み」。

### ボス

`examDay` が null →「日程未定」、今日 ≤ `examDay` → 残り日数を表示、今日 > `examDay` かつ
`result` が `pending` → 結果の入力を促す。

## 計算で求める値（保存しない）

すべて `src/domain/` の純粋関数で求め、引数に「今の時刻」と上のデータを受け取る。

| 値 | 求め方 | 関数の置き場所 |
|---|---|---|
| 今日の学習日・今週の範囲 | 現地時刻 − 4時間。週は月曜始まり | `dates.ts` |
| 今日の復習 | 出題日 ≤ 今日のメモを古い順に最大50件 | `reviewQueue.ts` |
| 就活の日 | 予定名のキーワード判定 ＋ 手動の切り替え | `jobHunt.ts` |
| スキマ時間 | 範囲内で予定と重ならない30分以上の時間帯 | `jobHunt.ts` |
| デイリー／ウィークリーの目標と進み具合 | R11 の式。進み具合はメモ・回答履歴・勉強の記録から集計 | `quests.ts` |
| 経験値・レベル・称号 | R14 の点数表で全履歴を合計 | `progress.ts` |
| 連続日数 | 条件を満たす日が今日（または昨日）から何日続くか | `progress.ts` |
| 地方の状態 | 中分類ごとのメモ数と直近30日の「覚えてた」の割合 | `worldMap.ts` |
| 用語の重複 | `termKey` が同じメモがあるか | `duplicate.ts` |

## 端末だけに置くもの（localStorage）

| キー | 中身 | 使い道 |
|---|---|---|
| `fapq.outbox` | 送信待ちの書き込み（メモ・記録・プロフィール） | 通信がないときの保存（R4） |
| `fapq.snapshot` | 前回表示したプロフィールとメモの写し | 起動直後の表示（SC-002） |
| `fapq.ui` | 最後に開いた画面など | 使い勝手のため |

予定の中身はここにも保存しない。
