# PORTING.md — お引越し可能ファイル

`RFrontEnd`傘下プロジェクト間、および他のRust系エコシステム
(open-raid-z/aruaru-db/open-web-server等)へそのまま(または軽微な
変更で)移植できる実装パターン一覧。

## `TokenSink`パターン(RHTML由来)

トークナイザ(字句解析器)とDOM木構築器を疎結合にする設計。
`TokenSink`トレイトを実装するだけで、トークナイザ側のコードを
一切変更せずに別の消費側(DOM構築・デバッグ用収集器等)へ差し替えられる。

```rust
pub trait TokenSink {
    fn process_token(&mut self, token: Token);
}
```

新しい構文解析器(将来のRTypeScript等)を書く際も、字句解析と
構文木構築を同じパターンで分離しておくと、テスト・デバッグ・
将来の再実装がしやすくなる。

## DOM実装非依存のマッチングトレイト(RCSS由来)

CSSエンジン(`RCSS`)は特定のDOM実装(`RHTML::Element`)に直接
依存せず、`ElementLike`トレイト(`tag_name()`/`classes()`/`id()`)
経由で要素とマッチングする。

```rust
pub trait ElementLike {
    fn tag_name(&self) -> &str;
    fn classes(&self) -> Vec<&str>;
    fn id(&self) -> Option<&str>;
}
```

この設計により、CSSエンジン自体を別のUIツールキット・別のDOM表現
(将来のネイティブUI等)にもそのまま応用できる。「エンジンAとエンジンB
を組み合わせて使いたいが、片方が片方の内部実装に直接依存してしまう」
という結合を避けたい場面全般で応用可能。

## `Patch`ベースの差分表現(RReact由来)

仮想DOMの差分計算結果を「置き換え」「テキスト更新」「要素の
in-place更新(属性・子要素どちらか一方または両方)」という
明示的な列挙型で表現する設計。

```rust
pub enum Patch {
    Replace(VNode),
    UpdateText(String),
    UpdateElement { attrs: Option<AttrsPatch>, children: Option<Vec<ChildPatch>> },
    NoOp,
}
```

「属性差分と子差分の両方がある場合の表現」を後から無理やり
組み合わせるのではなく、最初から複合ケースを持つ専用バリアントとして
設計しておくと、呼び出し側のパターンマッチが素直になる(このリポジトリ
自身が最初の実装で複合ケースの表現に迷い、書き直した実例がある——
`RReact/CLAUDE.md`のHANDOFF参照)。

## `rust_json`の型付き厳密モードAPI(RJSON由来)

`serde_json::from_str::<T>`相当の型付きデシリアライズを、
このエコシステム共通のJSON処理クレート(`RJSON`)経由に統一するための
関数群(`from_slice_strict::<T>`/`from_str_strict::<T>`/
`to_vec_strict`/`to_string_strict`/`to_string_pretty_strict`)。
`serde_json`へそのまま委譲するため出力バイト列は完全に同一——
チェックサムに依存する既存コード(`aruaru-db`のZFS互換チェックサム層等)
への移行時にも安全に使える。

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
