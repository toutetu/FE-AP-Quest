# Implementation Plan: 第1弾 メモ・復習・クエスト

**Branch**: `001-memo-review-quest` | **Date**: 2026-10-06 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-memo-review-quest/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

過去問道場で調べた用語を10秒でメモし、1日 → 3日 → 7日 → 14日 → 30日 → 60日の間隔で
自動的に復習できるようにする。学習の管理は、デイリー／ウィークリークエスト、ボス戦（試験日）、
経験値とレベル、図鑑、ワールドマップでゲーム調に見せる。Googleカレンダーの面接・面談を見て、
その日のノルマを軽くし、週の中で割り振り直す。

技術面では、TypeScript と React で1ページのアプリを作り、claude.ai の非公開ページ（Artifact）として
公開する。記録は Artifact の `db`（本人だけが読める領域）、AIは `sample`、カレンダーは `mcp` を使い、
自前のサーバーやAPIキーは持たない。復習日・ノルマ・経験値などの計算は画面から切り離した純粋関数にし、
Vitest で自動テストする。詳細な判断は [research.md](./research.md) にまとめた。

## Technical Context

**Language/Version**: TypeScript 7.0（strict）

**Primary Dependencies**: React 18.3.1（cdnjs の UMD 版をページから読み込む）、Vite 8、
vite-plugin-singlefile 2.3（ビルド時のみ）

**Storage**: claude.ai Artifact の `db` ケーパビリティ（`data/users/<本人のid>/` 以下の非公開領域）。
端末内の `localStorage` は、未送信メモの一時保管と、起動を速くするための表示キャッシュにだけ使う

**Testing**: Vitest 5（学習ロジックの純粋関数）。画面は手動確認（スマホ幅375px・PC幅、ライト・ダーク）

**Target Platform**: claude.ai の Artifact ビューア（スマホのブラウザ・アプリ、PCのブラウザ・デスクトップアプリ）

**Project Type**: 1ページのWebアプリ（SPA）。1つのHTMLにまとめて公開する

**Performance Goals**: アプリを開いてから今日やることが表示されるまで3秒以内。
メモの保存操作が10秒以内に終わる画面構成

**Constraints**: 自前サーバー・APIキーなし。外部への通信は `window.claude` の機能だけ
（ページのCSPにより、許可されたCDNのスクリプトとGoogle Fontsのほかは読み込めない）。
`alert` / `confirm` は使えないため、確認はページ内の画面で行う。日本語UI、ライト・ダーク両対応

**Scale/Scope**: 利用者1人。メモは12月までに1,000〜3,000件、勉強の記録は数百件。
画面は7つ（ホーム、メモ入力、復習、図鑑、ワールドマップ、取り込み、設定）

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原則 | 確認内容 | 結果 |
|---|---|---|
| I. 合格ファースト | 仕様の範囲だけを作り、演習・擬似言語・記述添削は第2弾以降に回している | 合格 |
| II. 毎日続く・すぐ使える | メモは用語と説明の2項目で保存でき、ホームの一番上に入力欄がある。就活の日も「復習だけ」の最低ラインを示す | 合格 |
| III. プライバシーを守る設計 | 記録は本人だけが読める領域に置く。予定は画面表示とノルマ計算にだけ使い、保存しない。カレンダーのアカウント名は本人の設定にだけ保存し、リポジトリに書かない | 合格 |
| IV. 仕様が先、小さく出す | 本計画を経てからタスク化する。使う場面の優先度（P1 → P3）の順に、それぞれ単独で使える状態で積み上げる | 合格 |
| V. 学習ロジックはテストで守る | 復習日・復習の順番・ノルマの割り振り・就活判定・スキマ時間・経験値・連続日数・マップの状態・取り込みの解析を `src/domain/` の純粋関数にし、Vitest で検証する | 合格 |
| VI. AIの答えは疑える形で出す | AIの結果に「AIが作成」の印を付けて保存し、採用・修正・削除を利用者が選ぶ | 合格 |
| 技術上の制約 | TypeScript、Artifactでの公開、サーバーなし、375px対応、GitHubで管理 | 合格 |

**設計後の再確認（Phase 1 のあと）**: [data-model.md](./data-model.md) と [contracts/](./contracts/) を
確認し、違反がないことを確かめた。経験値や連続日数を「数え上げる値」として保存せず、メモと勉強の記録から
毎回計算する設計にしたことで、端末間の書き込みの食い違いも起きにくくなった（research.md R5）。

## Project Structure

### Documentation (this feature)

```text
specs/001-memo-review-quest/
├── plan.md              # 本ファイル
├── research.md          # Phase 0: 技術的な判断とその理由
├── data-model.md        # Phase 1: 保存するデータと計算で求める値
├── quickstart.md        # Phase 1: 動作確認の手順
├── contracts/           # Phase 1: 外部とのつなぎ口と画面の決まりごと
│   ├── ports.md         #   保存・AI・カレンダーのインターフェース
│   ├── ai-check.md      #   AIチェックの指示文と返答の形
│   ├── calendar.md      #   Googleカレンダーの呼び出しと変換
│   └── screens.md       #   画面とURLの対応
├── checklists/
│   └── requirements.md  # 仕様の品質チェック
└── tasks.md             # Phase 2（/speckit-tasks で作成）
```

### Source Code (repository root)

```text
index.html                 # 開発用の入口（Vite）
vite.config.ts
tsconfig.json
package.json
scripts/
└── build-artifact.mjs     # ビルド結果を Artifact 用の1ファイルに整える

src/
├── main.tsx               # 起動。window.claude があれば claude 用、なければ開発用のつなぎ口を選ぶ
├── domain/                # 純粋関数（画面にも外部にも依存しない。すべてテスト対象）
│   ├── dates.ts           #   学習日（午前4時区切り）、週（月曜始まり）
│   ├── fields.ts          #   IPAの分野（大分類9・中分類23）
│   ├── srs.ts             #   間隔反復（段階と次の出題日）
│   ├── reviewQueue.ts     #   今日の復習（1日50件まで、古い順）
│   ├── jobHunt.ts         #   就活の日の判定、スキマ時間
│   ├── quests.ts          #   デイリー／ウィークリーの目標と割り振り直し
│   ├── progress.ts        #   経験値、レベル、称号、連続日数
│   ├── worldMap.ts        #   地方の状態（霧・攻略中・制覇）
│   ├── duplicate.ts       #   用語の正規化と重複の判定
│   └── importParser.ts    #   貼り付けた文章をメモに分ける
├── ports/                 # 外部とのつなぎ口（インターフェースだけ）
│   ├── storage.ts
│   ├── ai.ts
│   └── calendar.ts
├── adapters/
│   ├── claude/            # db / sample / mcp を使う本番用の実装
│   │   ├── dbStorage.ts
│   │   ├── sampleAi.ts
│   │   └── mcpCalendar.ts
│   └── local/             # 開発用（localStorage とダミーの返答）。第2弾以降のデモ版でも使う
│       ├── localStorage.ts
│       ├── fakeAi.ts
│       └── fakeCalendar.ts
├── app/                   # 状態の組み立て（購読、派生値の計算、未送信メモの再送）
│   ├── AppState.tsx
│   └── outbox.ts
└── ui/
    ├── App.tsx            # 画面の切り替え（#home など）
    ├── screens/           # Home, MemoForm, Review, Zukan, WorldMap, Import, Settings
    ├── components/        # QuestCard, BossBar, MonsterCard, ConfirmDialog など
    └── styles/
        └── tokens.css     # 色・文字のトークン（ライト・ダーク）

tests/
└── domain/                # src/domain の各ファイルに対応するテスト
```

**Structure Decision**: 1つのプロジェクト構成にした。学習ロジック（`src/domain/`）、外部とのつなぎ口
（`src/ports/` と `src/adapters/`）、画面（`src/ui/`）を分けることで、学習ロジックをテストしやすくし、
claude.ai の外（開発時やデモ版）でも同じ画面を動かせるようにする。

## Complexity Tracking

憲法に反する点はないため、記載なし。
