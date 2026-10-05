# Contract: 外部とのつなぎ口（ports）

画面と学習ロジックは、保存・AI・カレンダーを次のインターフェース越しにだけ使う。
本番は `src/adapters/claude/`（db / sample / mcp）、開発とデモは `src/adapters/local/` が実装する。
型の詳細は [data-model.md](../data-model.md) を参照。

## 共通の状態

```ts
/** つなぎ口が今使えるかどうか。画面はこれで表示を切り替える */
type PortStatus =
  | { kind: "ready" }
  | { kind: "unavailable"; reason: "signed-out" | "not-granted" | "disabled" }
  | { kind: "needs-action"; message: string }; // 例: カレンダーのつなぎ直し
```

## StoragePort

```ts
interface StoragePort {
  status(): Promise<PortStatus>;

  /** 購読は画面の起動時に1回だけ。戻り値で購読をやめる */
  subscribeProfile(next: (p: Profile) => void, onError: (e: StorageError) => void): () => void;
  subscribeMemos(next: (memos: Memo[]) => void, onError: (e: StorageError) => void): () => void;
  subscribeStudyLogs(next: (logs: StudyLog[]) => void, onError: (e: StorageError) => void): () => void;

  /** 新規も更新も同じ。id が同じなら上書き（再送しても重複しない） */
  saveMemo(memo: Memo): Promise<void>;
  deleteMemo(id: string): Promise<void>;
  saveStudyLog(log: StudyLog): Promise<void>;
  deleteStudyLog(id: string): Promise<void>;
  saveProfile(profile: Profile): Promise<void>;

  /** 端末側で新しい id を作る */
  newId(): string;
}

type StorageError =
  | { code: "quota"; message: string }        // 件数・容量の上限
  | { code: "busy"; message: string }         // 呼び出しが多すぎる。少し待つ
  | { code: "offline"; message: string }      // 一時的に届かない。送信待ちに残す
  | { code: "revoked"; message: string }      // このページを開いている間に権限がなくなった
  | { code: "invalid"; message: string };     // 実装の誤り
```

**db の error code との対応**: `quota_exceeded` → `quota`、`resource_exhausted` → `busy`、
`unavailable` → `offline`、`revoked` → `revoked`、`invalid_argument` / `transform_error` → `invalid`、
`not_granted` / `capability_disabled` / `capability_removed` → `status()` が `unavailable`。

**決まりごと**
- 同じ文書への書き込みは、前の書き込みが終わってから行う。
- 画面の描画や購読の通知の中から書き込まない。書き込むのは利用者の操作のときだけ。
- `saveMemo` が `offline` で失敗したら、送信待ち（`app/outbox.ts`）に残して後で再送する。

## AiPort

```ts
interface AiPort {
  status(): Promise<PortStatus>;
  /** 利用者がボタンを押したときだけ呼ぶ。signal で途中でやめられる */
  checkMemo(input: AiCheckInput, signal: AbortSignal): Promise<AiCheckOutcome>;
}

type AiCheckInput = {
  term: string;
  explanation: string;
  exam: "AP" | "FE" | null;
  fieldMinor: number | null;
};

type AiCheckOutcome =
  | { ok: true; result: AiResult }
  | { ok: false; reason: "hidden" }          // 使えない。ボタンを隠す
  | { ok: false; reason: "rate-limited" }    // しばらく待ってからもう一度
  | { ok: false; reason: "retry"; partial?: string } // もう一度押せる
  | { ok: false; reason: "cancelled" };
```

**sample の error code との対応**: `not_granted` / `sampling_disabled` / `not_declared` /
`capability_disabled` / `capability_removed` → `hidden`、`rate_limited` → `rate-limited`、
`invalid_json` / `upstream_error` / `empty_completion` / `refused` / `session_expired` → `retry`、
`cancelled` → `cancelled`。指示文と返答の形は [ai-check.md](./ai-check.md)。

## CalendarPort

```ts
interface CalendarPort {
  status(): Promise<PortStatus>;
  listCalendars(): Promise<CalendarChoice[]>;
  /** 範囲内の予定。予定の中身は呼び出し側で保存しない */
  listEvents(calendarId: string, range: { start: Date; end: Date }, opts?: { refresh?: boolean }): Promise<CalendarResult>;
}

type CalendarChoice = { id: string; name: string };

type CalendarEvent = {
  title: string;
  start: Date;        // 終日の予定はその日の0:00
  end: Date;
  allDay: boolean;
};

type CalendarResult =
  | { ok: true; events: CalendarEvent[]; fetchedAt: Date }
  | { ok: false; reason: "reconnect" | "not-allowed" | "unavailable" };
```

呼び出しと変換の詳細は [calendar.md](./calendar.md)。
