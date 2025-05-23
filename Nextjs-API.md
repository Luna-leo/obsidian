
### 原因

`next build` のあと **`out` フォルダだけを IIS に配置すると、Next.js は静的 HTML/JS だけの “エクスポート” モード** として動くため **API Routes (`/pages/api/*` や `/app/api/*`) は一切含まれません**。  
API Route は Node.js ランタイムで動的に動く機能なので、静的エクスポートでは 404 になります。Next.js 公式ドキュメントでも「`next export`（あるいは `output: 'export'`）では API Routes が無効になる」と明記されています。([Next.js](https://nextjs.org/docs/messages/api-routes-static-export "Understanding \"API Routes in Static Export\" Warning in Next.js | Next.js"))

---

### 対応パターン

| パターン                                                                                                                                   | 長所                       | 短所 / 注意                                   |
| -------------------------------------------------------------------------------------------------------------------------------------- | ------------------------ | ----------------------------------------- |
| **A. `next start` を動かす**1. `next build`2. `next start -p 11700` でアプリを常駐3. IIS (ARR + URL Rewrite など) で `http://server:11700` にリバースプロキシ | 現在のコードのまま API Route が使える | サーバーに Node.js が常駐する構成が必要                  |
| **B. iisnode を使う**pages ディレクトリごと IIS に配置し、`iisnode` で Node プロセスをホスト                                                                    | Windows/IIS ネイティブにまとまる   | iisnode はメンテがやや停滞気味、設定が複雑                 |
| **C. API を別サービス化**API Route 部分を Express / FastAPI / Azure Functions などに移してフロントエンドは純静的サイト (`out` フォルダ)                                  | IIS で単なる静的配信で済む          | フロントエンド側の fetch URL を別ドメイン / パスに変更する必要がある |
| **D. App Router + Route Handlers (Edge / Node)**Next 14 以降の `/app/{route}/route.js` でも、実行時に動くため結局 Node/Edge ランタイムが要る。静的配信だけでは動かない      | 最も新しい構文                  | 静的エクスポートではやはり 404                         |

---

### IIS で **A. `next start` を採用**する場合の最小手順

1. **ビルド**
    
    ```bash
    npm run build           # = next build
    ```
    
2. **環境変数を設定**
    
    - `PORT=11700`
        
    - SQL Server 接続文字列 (`MSSQL_CONN=...`) など  
        （IIS の「環境変数」または `web.config` の `<environmentVariables>` で設定）
        
3. **起動スクリプト**  
    バッチ or NSSM などで Windows サービス化して
    
    ```bash
    npm run start           # = next start -p 11700
    ```
    
4. **IIS でリバースプロキシ**
    
    - _Application Request Routing_ と _URL Rewrite_ を有効化
        
    - ルール例:
        
        ```
        <rule name="NextProxy" stopProcessing="true">
          <match url="(.*)" />
          <action type="Rewrite" url="http://localhost:11700/{R:1}" />
        </rule>
        ```
        
5. ブラウザーから `http://サーバー名/` へアクセス → `/api/*` も同一 Origin で 200 が返るはず。
    

---

### すぐに確認したいチェックリスト

1. **`next export` や `output: 'export'` が有効になっていないか？**
    
2. **IIS がポート 11700 の Node プロセスへ正しくプロキシしているか？**
    
3. **IIS サーバーに Node.js と必要な npm パッケージが入っているか？**
    
4. **環境変数 （SQL 接続文字列）を本番サーバーにも渡しているか？**
    

---

#### 次のステップ

- 「Node を常駐させるのは避けたい」「社内ポリシーでサービス常駐が難しい」など制約があれば、パターン C（API を別サーバーやサーバーレスへ切り出し）をご検討ください。
    
- どの構成を採るかまだ迷っている場合は、運用制約（常駐プロセス可否、ファイアウォール、社内 NW ポリシー）を共有いただければ、より具体的な手順をご提案できます。