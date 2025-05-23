以下は **SQL Server 認証** を使って Next .js の API ルートからデータを取得する最小構成サンプルです。  
（Next .js 14／App Router を想定。Pages ルーティングを使っている場合は最後に載せた「Pages 版」を参照してください）

---

## 1. 依存関係を追加

```bash
npm install mssql        # SQL Server ドライバ
npm install --save-dev @types/mssql  # TypeScript 補完が欲しい場合
```

---

## 2. 環境変数を設定（`.env.local`）

```text
SQL_SERVER_HOST=localhost       # サーバー名 / IP
SQL_SERVER_PORT=1433            # 既定 1433
SQL_SERVER_DATABASE=MyDatabase
SQL_SERVER_USER=myuser
SQL_SERVER_PASSWORD=mysecret
```

> ⚠️ 認証情報は **絶対に Git 管理しない** よう `.gitignore` を確認してください。

---

## 3. 共有コネクションプールを用意 `lib/db/mssql.ts`

```ts
// lib/db/mssql.ts
import sql, { config as SqlConfig, ConnectionPool } from 'mssql';

const cfg: SqlConfig = {
  server: process.env.SQL_SERVER_HOST!,
  port: Number(process.env.SQL_SERVER_PORT ?? 1433),
  database: process.env.SQL_SERVER_DATABASE!,
  user: process.env.SQL_SERVER_USER!,
  password: process.env.SQL_SERVER_PASSWORD!,
  options: {
    encrypt: true,              // Azure / TLS 接続時は true
    trustServerCertificate: true// 社内検証用 TLS なら true で回避
  },
  pool: { max: 10, min: 0, idleTimeoutMillis: 30_000 }
};

// Hot-reload 時にプールを使い回すためグローバル保持
let pool: ConnectionPool | null = null;

export async function getPool(): Promise<ConnectionPool> {
  if (pool) return pool;              // 生成済み
  pool = await sql.connect(cfg);      // 初回のみ接続
  return pool;
}
```

---

## 4. API ルートを作成 `app/api/employees/route.ts`

```ts
// app/api/employees/route.ts
import { NextResponse } from 'next/server';
import sql from 'mssql';
import { getPool } from '@/lib/db/mssql';

export async function GET() {
  try {
    const pool = await getPool();

    // ★サンプルクエリ：従業員テーブル先頭 100 件
    const result = await pool
      .request()
      .query('SELECT TOP (100) id, name FROM dbo.Employees ORDER BY id');

    return NextResponse.json(result.recordset); // => [{ id: 1, name: '...' }, ...]
  } catch (err) {
    console.error('DB error', err);
    return NextResponse.json({ message: 'Database error' }, { status: 500 });
  }
}
```

### パラメータ付きクエリ例（SQL インジェクション対策）

```ts
export async function GET(req: Request) {
  const { searchParams } = new URL(req.url);
  const employeeId = Number(searchParams.get('id'));  // /api/employee?id=123

  const pool = await getPool();
  const result = await pool
    .request()
    .input('id', sql.Int, employeeId)                 // 型を指定してバインド
    .query('SELECT * FROM dbo.Employees WHERE id = @id');

  return NextResponse.json(result.recordset[0] ?? null);
}
```

---

## 5. （Pages ルーティングを使う場合）

```ts
// pages/api/employees.ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { getPool } from '../../lib/db/mssql';

export default async function handler(
  _req: NextApiRequest,
  res: NextApiResponse
) {
  try {
    const pool = await getPool();
    const result = await pool
      .request()
      .query('SELECT TOP (100) id, name FROM dbo.Employees ORDER BY id');
    res.status(200).json(result.recordset);
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'DB error' });
  }
}
```

---

### ワンポイント

|項目|説明|
|---|---|
|**プール共有**|API ルートは毎リクエストごとにロードされるため、`getPool()` でプールを再利用します。|
|**TLS 設定**|本番環境で自己署名証明書を使わない場合は `trustServerCertificate` を `false` に。|
|**Windows 認証**|Node.js から Windows 認証 (Integrated Security) を使う場合は `msnodesqlv8` ドライバに切り替えが必要です。|

> 以上で、Next .js 側から SQL Server へ安全・効率的にアクセスできます。  
> クエリや戻り値を変えれば、そのまま FastAPI バックエンドから取得した JSON をフロントエンドへ渡す構成にも流用できます。