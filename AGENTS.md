# Agent Instructions

## Purpose

You will help the user build a React web app that visualizes data from Power BI semantic models. The app fetches live data via DAX queries, renders charts and grids using Vega-Lite and a built-in DataGrid component, and supports light/dark theming. Your job is to discover the user's data, write correct DAX queries, build React components that fetch and display that data, and validate the result in the browser.

### Project Documents

Before implementation, always read:

- REQUIREMENTS.md
- UI_SPEC.md
- MODEL_CONFIG.md

Use these files as the source of truth.

Priority order:

1. MODEL_CONFIG.md
2. REQUIREMENTS.md
3. UI_SPEC.md

Do not infer missing requirements.


## Semantic model schema (discover it progressively)

Before the first schema command, ensure the user- or task-provided semantic model is registered and
its connection alias is known. Never register an inferred or additional data source.

Discover the connected model's schema on demand with DAX `INFO.VIEW.*` queries run through `npx fabric-app-data query`. Fetch only what the current request needs: start with a scope probe, then tables, then columns and measures for the tables that are relevant.

Do **not** fetch the whole schema upfront, and do **not** rely on `src/fabric.generated.ts` for schema because it holds only connection aliases.

Complete the initial semantic-model discovery with the supplied connection details and model
metadata before inspecting generic application source files. Do not include `global.css`,
`package.json`, `App.tsx`, hooks, utilities, or broad `src/**` patterns in initial glob, search, or
read operations. After initial discovery, inspect only the files needed to implement the request.
When modifying existing behavior, relevant query files may be inspected earlier.

The [schema-discovery](.agents/skills/schema-discovery/SKILL.md) skill has the discovery order, INFO function map, and narrowing patterns.

## Runtime rules

### DAX結果列名と表示用列名の対応

- DAXに記述した列名と、SDKが返す結果列名を同一とみなさない。
- このアプリのSELECTCOLUMNSによる別名列は、結果では
  `[Year]`、`[SalesAmount]`などの角括弧付きの名前になる。
  モデル列の出力には`Date[Year]`などの名前もあるため、
  一律に角括弧を付けたり除去したりせず、実際の結果を確認する。
- `columnMetadata`のキーは、成功結果の`data.table.columns[].name`
  と完全一致させる。`name`には角括弧を含まない表示用の別名を指定する。

正しい例：

```ts
columnMetadata: {
    "[Year]": { name: "year", displayName: "年" },
    "[SalesAmount]": {
        name: "salesAmount",
        displayName: "売上金額",
        format: "#,##0",
    },
}
```

- Vega-Liteのフィールド名とデータ整形処理が参照する列名は、
  `toDataTable`で変換した後の`name`に統一する。
- この確認はグラフだけでなく、フィルター候補取得クエリにも適用する。
- テストには表示用の別名だけでなく、SDK結果の生の列名を入力し、
  `toDataTable`によって期待する別名へ変換されることを検証する。
- 型チェック・ビルド・単体テストの成功だけでは、
  実際の結果列名が正しいことを確認した扱いにしない。
- 認証等で実クエリを検証できない場合は未検証と明記し、
  スキーマ確認済み・実データ検証済みと報告しない。


## API discovery

Use exact symbol searches and the narrowest relevant public declaration or template-local source file to learn an API. Do not broadly list, search, read, or dynamically import package internals, compiled JavaScript bundles, or Vite dependency bundles. Read runtime implementation only when the task explicitly requires debugging behavior.

For visualizations backed by a Fabric semantic model, follow these four steps:

```tsx
// 1. barrel   src/queries/<page>/<viz>.ts exports { connection, query, columnMetadata, vegaLiteSpec }
// 2. fetch    const { data, isLoading, error } = useSemanticModelQuery({ connection, query });
// 3. shape    const table = toDataTable(data.table, columnMetadata);
// 4. render   <VegaVisual spec={vegaLiteSpec} data={table} theme={theme} />   // or <DataGrid data={table} theme={theme} />
```

When a component uses `useThemeContext()` or `useSemanticModelQuery()`, call the hook unconditionally at the top of the component before any loading or error return.

Runtime rules:

- **DAX failures do not throw from the SDK.** `useSemanticModelQuery` maps failed query results, transport failures, and unexpected runtime failures to its `error` field. Handle `error`, then require `data.status === "success"` before touching `data.table`.
- Key `columnMetadata` by the query's exact raw output column name. Give each entry a bracket-free `name` alias for Vega-Lite and DataGrid fields; raw names containing `.`, `[`, or `]` can render as `undefined`.
- Pass the resolved `theme` from `useThemeContext()` to `VegaVisual` and `DataGrid`.

## Project structure

```
fabric.yaml                # Fabric connection config
index.html                 # Vite entry HTML
vite.config.ts             # Vite + Tailwind build config
tsconfig.json              # TypeScript configuration
src/
├── fabric.generated.ts    # Connection aliases to workspace/item IDs
├── main.tsx               # App entry point
├── App.tsx                # Main dashboard layout
├── ErrorFallback.tsx      # Error boundary fallback UI
├── global.css             # Tailwind design tokens
├── data-palette-presets.json
├── components/
├── hooks/
├── lib/
├── queries/               # DAX, Vega-Lite specs, and factory functions
└── vite-env.d.ts
```

## Query and spec organization

Group query files by page or domain under `src/queries/`. Each visualization uses the same kebab-case base name for its `.dax`, `.json`, and `.ts` files.

- Keep all DAX in `.dax` files and import it with Vite's `?raw` suffix.
- Use separate `.dax` files when a parameter changes query structure.
- Export a factory returning `{ connection, query, columnMetadata, vegaLiteSpec }`.
- Key `columnMetadata` by exact query output names and map each to a Vega-safe `name`, user-facing `displayName`, and optional `format`.
- Re-export modules through an `index.ts` in each group and at the `src/queries/` root.
- Never define Vega-Lite specs inline in component files.

Validate each single-table visual immediately after completing its query factory and before UX design, editing `src/global.css`, or writing component code. Run `npm run validate:visual -- <query-factory.ts> [factory-params-json] [<query-factory.ts> [factory-params-json] ...]`, repeating a factory path for every relevant parameter variant. Run the validation command directly without piping it through `tail`, `grep`, `head`, or another command that can hide a nonzero exit status. Validate all completed factories in one command when possible, and do not proceed until every result passes.

After the required schema and query results are understood, load and follow the [app-design](.agents/skills/app-design/SKILL.md) skill before writing presentation code.

## Validation

Browser validation is required after UI changes. After implementation is complete, load and follow the [app-validation](.agents/skills/app-validation/SKILL.md) skill, validate through the Fabric portal embed flow, and fix issues before considering the task complete. Use only the target Fabric workspace URI supplied by the user or task. If deployment configuration is missing and no target workspace URI was supplied, ask the user for one.

## Critical rules

1. **Never use mock, fake, or hardcoded data.** All data must come from a real source.
2. **Never store data in memory or local storage.** Fetch on demand from the real source.
3. **Do not assume or silently add a data source.** Confirm the source with the user and never supplement it without explicit consent.
4. **Never guess query result schema.** Run the query first and use its exact output names.
5. **Do not ask the user to describe the schema.** Discover it with DAX `INFO.VIEW.*` queries.
