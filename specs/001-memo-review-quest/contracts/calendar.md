# Contract: Googleカレンダー

`mcp` ケーパビリティで、claude.ai の「Google Calendar」コネクタを利用者本人の権限で呼ぶ。
公開時の宣言: `mcp: { servers: [{ server: "Google Calendar", tools: ["list_calendars", "list_events"] }] }`

## カレンダーの一覧（設定画面）

```ts
callTool("Google Calendar", "list_calendars", {})
```

返答の `payload.calendars[]` から `{ id, summary }` を取り出し、`{ id, name: summary }` として選択肢に出す。
選んだ `id` は profile の `calendarId` にだけ保存する。

## 今週の予定

```ts
callTool("Google Calendar", "list_events", {
  calendarId,                       // profile.calendarId
  startTime: "2026-10-05T04:00:00+09:00",   // 今週の月曜4:00
  endTime:   "2026-10-12T04:00:00+09:00",   // 翌週の月曜4:00
  timeZone: "Asia/Tokyo",
  orderBy: "startTime",
  pageSize: 250,
}, { cache: { staleTime: 300000, gcTime: 3600000 } })
```

- 返答の `payload.nextPageToken` があれば、`pageToken` を付けて最大3回まで続きを取る。
- ホームの「更新」ボタンでは `cache: { refresh: true }` を付けて取り直す。

## 返答の変換

実際の呼び出しで、`payload` が次の形であることを確かめた（値は架空の例）。

```json
{
  "events": [
    { "summary": "サンプル株式会社 一次面接", "start": { "dateTime": "2026-10-07T14:00:00+09:00" }, "end": { "dateTime": "2026-10-07T15:00:00+09:00" } },
    { "summary": "終日の予定", "start": { "date": "2026-10-10" }, "end": { "date": "2026-10-11" } }
  ],
  "nextPageToken": "..."
}
```

| 返答 | CalendarEvent |
|---|---|
| `summary`（なければ空文字） | `title` |
| `start.dateTime` があれば その日時、なければ `start.date` の 0:00 | `start` |
| `end.dateTime` があれば その日時、なければ `end.date` の 0:00 | `end` |
| `start.date` だけのとき | `allDay: true` |

形が合わない予定（日付が読めないなど）は捨てる。変換は `adapters/claude/mcpCalendar.ts` で行い、
domain には `CalendarEvent[]` だけを渡す。

## エラーの扱い

| code | CalendarResult | 画面 |
|---|---|---|
| `needs_reauth` | `reconnect` | 「claude.ai の設定 → コネクタで Googleカレンダーをつなぎ直してください」 |
| `server_not_connected` / `selection_required` / `server_not_found` | `reconnect` | 同上（「追加してください」） |
| `not_in_manifest` / `blocked_by_policy` / `approval_required` | `not-allowed` | 「カレンダーを使わない設定です」。何度も聞かない |
| `server_unavailable` / `rate_limited` | `unavailable` | 1回だけ `retryAfterMs`（なければ2〜5秒）待って取り直し、だめなら小さく表示 |
| `not_granted` / `capability_disabled` / `capability_removed` | `status()` が `unavailable` | カレンダーの欄を出さない |
| その他・`tool_error` | `unavailable` | 小さく表示 |

どの場合も、ノルマは「就活の日なし」として計算し、アプリはそのまま使える（ストーリー5の受け入れ条件5）。
