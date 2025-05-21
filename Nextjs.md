
以下の手順とサンプルコードをベースにすると、

- **ヘッダー３段（多階層ヘッダー）**
    
- **左２列をインデックスとして固定（ピン留め）**
    
- **セル編集**
    

をすべて Next.js ＋ MUI X Data Grid Pro で実装できます。

---

## 1. パッケージの準備

```bash
# Data Grid Pro を使う場合（v6 以上を推奨）
npm i @mui/x-data-grid-premium @mui/material @mui/icons-material @emotion/react @emotion/styled
# Premium が必要なら pro → premium に読み替えてください
```

> **ライセンスキー**  
> 商用利用の場合は `LicenseInfo.setLicenseKey('YOUR_KEY')` をどこかアプリ初期化時に呼び出してください。

---

## 2. Next.js で “クライアント専用” に読み込む

Data Grid はブラウザ API を使うので **SSR を無効** 化した動的 import が無難です。

```tsx
// components/SensorTable.tsx
'use client';                           // App Router の場合は必須
import dynamic from 'next/dynamic';
import { Box } from '@mui/material';

// DataGridPro だけクライアント側でロード
const DataGridPro = dynamic(
  () => import('@mui/x-data-grid-pro').then((m) => m.DataGridPro),
  { ssr: false }
);
```

---

## 3. カラム定義・グループ階層（＝ヘッダー３行）

```ts
// 2 本のインデックス列（編集不可 & ピン留め）
const indexColumns = [
  { field: 'plant', headerName: 'プラント', width: 140, editable: false },
  { field: 'machine', headerName: '機械',  width: 120, editable: false },
];

// センサ列（編集可）
const dataColumns = [
  { field: 'tempA', headerName: 'センサ A', width: 120, type: 'number', editable: true },
  { field: 'tempB', headerName: 'センサ B', width: 120, type: 'number', editable: true },
  { field: 'humA',  headerName: '湿度 A',  width: 120, type: 'number', editable: true },
];

// 3 段ヘッダーを columnGroupingModel で表現
const columnGroupingModel: GridColumnGroupingModel = [
  {
    groupId: 'env',          // ── 1 段目 ──
    headerName: '環境情報',
    children: [
      {
        groupId: 'temp',     // ── 2 段目 ──
        headerName: '温度',
        children: [{ field: 'tempA' }, { field: 'tempB' }], // ── 3 段目 ──
      },
      {
        groupId: 'hum',
        headerName: '湿度',
        children: [{ field: 'humA' }],
      },
    ],
  },
];
```

ポイント

- `experimentalFeatures={{ columnGrouping: true }}` を付けると **多段ヘッダー** が有効になる ([MUI](https://mui.com/x/react-data-grid/column-groups/?utm_source=chatgpt.com "Data Grid - Column groups - MUI X"), [v6.mui.com](https://v6.mui.com/x/react-data-grid/column-groups/?utm_source=chatgpt.com "Data Grid - Column groups - MUI X"))
    
- 3 段まででもネストは自由。`field` を置いた場所が最下段セルになります。
    

---

## 4. 行データとピン留め設定

```ts
const rows = [
  { id: 1, plant: '東京第一', machine: 'M-001', tempA: 25.1, tempB: 24.8, humA: 42 },
  { id: 2, plant: '東京第一', machine: 'M-002', tempA: 26.3, tempB: 25.9, humA: 40 },
  // …
];

// index 列を常時表示
const pinnedColumns = { left: ['plant', 'machine'] };   // Data Grid Pro 6+
```

ピン留め（“frozen”）は `pinnedColumns` か `initialState.pinnedColumns` で指定します。  
詳細は公式ドキュメントを参照 ([MUI](https://mui.com/x/react-data-grid/column-pinning/?utm_source=chatgpt.com "Data Grid - Column pinning - MUI X"))

---

## 5. 完整コンポーネント

```tsx
// components/SensorTable.tsx
'use client';
import * as React from 'react';
import dynamic from 'next/dynamic';
import { Box } from '@mui/material';
import { GridRowsProp, GridValidRowModel } from '@mui/x-data-grid-pro';

const DataGridPro = dynamic(
  () => import('@mui/x-data-grid-pro').then((m) => m.DataGridPro),
  { ssr: false },
);

// ① columns / columnGroupingModel / rows は前項までのコードを貼り付け
//   ここでは import されたと仮定します

export default function SensorTable() {
  const [tableRows, setTableRows] = React.useState<GridRowsProp>(rows);

  // 編集後にローカル state を更新（API 連携する場合はここで fetch）
  const processRowUpdate = React.useCallback(
    (newRow: GridValidRowModel) => {
      setTableRows((prev) =>
        prev.map((r) => (r.id === newRow.id ? newRow : r)),
      );
      return newRow;
    },
    [],
  );

  return (
    <Box sx={{ height: 600, width: '100%' }}>
      <DataGridPro
        rows={tableRows}
        columns={[...indexColumns, ...dataColumns]}
        columnGroupingModel={columnGroupingModel}
        experimentalFeatures={{ columnGrouping: true }}
        pinnedColumns={pinnedColumns}
        editMode="cell"
        processRowUpdate={processRowUpdate}
        onProcessRowUpdateError={(err) => console.error(err)}
        disableRowSelectionOnClick
      />
    </Box>
  );
}
```

---

## 6. カスタマイズのヒント

|機能|使うプロパティ|備考|
|---|---|---|
|行単位編集|`editMode="row"` と `processRowUpdate`|行編集に切り替えるだけ|
|行/セルのバリデーション|`preProcessEditCellProps`|入力前に値を検証|
|独自ヘッダーのスタイル|`sx` や `classes`|`.MuiDataGrid-columnHeader--group` を上書き|
|行グループ表示（ツリー形インデックス）|`rowGroupingModel`|Premium 機能|

---

### まとめ

- **columnGroupingModel** で 3 行ヘッダーを作成
    
- **pinnedColumns.left** で左側 2 列を固定
    
- **editMode** と **processRowUpdate** で編集を有効化
    
- **dynamic import (ssr:false)** で Next.js と衝突しない
    

これで “三層ヘッダー＋左インデックス固定＋編集” の要件を一気に満たせます。さらに詳細な振る舞い（セルバリデーションや API 連携）を追加したい場合は、`processRowUpdate` 内で自由にロジックを挟んでください。



```planetext
DuckDBに登録されているデータを取得して画面上に表示したい。
DuckDBに登録されているデータ
parameter_id_master テーブル
- param_id　TEXT
- param_name　TEXT
- param_name_ja　TEXT
- plant_name　TEXT
- machine_no　TEXT
- data_source　TEXT
- insert_date　TIMESTAMP
PRIMARY KEY( param_id, plant_name, machine_no, data_source )

このテーブルからデータを取得する。
取得後縦持ちデータを横持ちデータに変形する。
カラムヘッダーはplant_name, machine_no, data_source　の３行のヘッダーとしたい。
行インデックスは、param_name_en, param_name_ja の２列として固定したい。
データとして表示する内容はparam_idを表示したい。
```



FastAPI（Python）でDuckDBからデータ取得して、以下内容でUI上に表示する
- ヘッダー３行（plant_name, machin_no, data_source
- インデックス列２列（param_name_en, param_name_ja）
- dataはparam_idを表示する。

 DuckDBのテーブル構成（parameter_id_master テーブル）は以下
- param_id　TEXT 
- param_name_en　TEXT 
- param_name_ja　TEXT 
- plant_name　TEXT 
- machine_no　TEXT 
- data_source　TEXT 
- insert_date　TIMESTAMP
PRIMARY KEY( param_id, plant_name, machine_no, data_source )

UIはMUI data gridを用いてテーブル表示したい。  （premiumのライセンスあり）
バックエンドの処理は仮実装として、ダミーデータでＡＰＩから取得したとして、
フロント側の処理を実装してほしい。

### データ選択UIを作成したい
取得したいデータの条件指定方法は以下の２パターンが選択でき、両方併用可能とする。
２パターンでの指定が完了したら、データ取得ボタンを押してデータ取得する。
- DBに登録されているイベント情報から条件指定する
	- データベースのイベントから、plant_name, machine_no, label, label_description, event, event_detail, start_time, end_timeをテーブル形式で表示。
	- 表示されているイベントの中から利用したいデータをチェックボックスにチェックを入れて選択
- 自分で指定する
	- plant_name, machine_no, start_time, end_timeを自分で指定する。
	- 複数条件指定可能とする




### センサーデータのデータベース化方針
CSVの時系列センサーデータをparquetファイル化することにより高圧縮化する。
またクエリパフォーマンスの向上が見込める。
データべース化の効率化やデータ取得時の利便性向上のためにメタ情報やマスタ情報を管理する。
管理するデータ一覧
-  ユニットタグマスター:DuckDB
	- 全プラント/号機のセンサー名の表記揺れ、センサーIDのマッピング
- センサーIDマスター:DuckDB
	- 各プラント/号機のセンサーID、項目名、単位を管理
- イベントマスター:DuckDB
	- 全プラント/号機のデータラベル/イベント情報を管理
-  センサーデータ:Parquet
	- 各プラント/号機のセンサーデータをParquet形式で保管
	- パーティショニングは plant_name / machine_no / month

UIから各情報の閲覧やデータベース化登録、データ取得、グラフ化ができるようにする
