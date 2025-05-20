# DataFrame の pivot でインデックスを消す方法

## pandas の場合（`param_name` を行インデックスにせず、普通の列にする）

```python
df_wide = (
    df.pivot_table(
        index="param_name",
        columns=["plant_name", "unit", "data_source"],
        values="param_id",
        aggfunc="first",
    )
    .reset_index()             # インデックスを列に戻す
    .rename_axis(None, axis=1) # 列名のインデックス名を消す（任意）
)
# param_name が不要なら .reset_index(drop=True)
```

- `reset_index()` …… 行インデックスだった `param_name` が普通の列に移動し、デフォルトの整数インデックスが付く
- `rename_axis(None, axis=1)` …… 列ヘッダー上部の "plant_name|unit|data_source" という名前（`columns.name`）を消したいときに使います

---

## Polars の場合

Polars には行インデックスの概念がないため、上記 `pivot()` を実行した時点で `param_name` はただの列になります。追加の操作は不要です。

```python
wide = (
    pl.from_pandas(df)
      .pivot(
          index="param_name",
          columns=["plant_name", "unit", "data_source"],
          values="param_id",
          aggregate_function="first",
      )
)
# wide は行番号 0,1,2... の DataFrame になる
```

---

## まとめ

- **pandas** では `pivot_table` のあと `reset_index()` を呼ぶと行インデックスを無くせる
- **Polars** ではインデックスが無いためそのまま

これで `param_name` が列として残り、行はデフォルトの整数インデックスになります。



以下は **Python の DataFrame（Pandas / Polars）を DuckDB のテーブルに登録する最小構成サンプル** です。  
ポイントは **① DataFrame を DuckDB に “ビュー” として登録 → ② `CREATE TABLE … AS SELECT …` または `INSERT INTO …`** の 2 段階で行うことです。

---

## 1. Pandas DataFrame を登録する例

```python
import pandas as pd
import duckdb

# ---- 0. サンプル DataFrame -------------------------------------------------
df = pd.DataFrame(
    {
        "plant_name": ["Plant-A", "Plant-B"],
        "machine_no": [101, 202],
        "temperature": [78.3, 81.5],
        "datetime": pd.to_datetime(["2025-05-19 12:00", "2025-05-19 12:05"]),
    }
)

# ---- 1. DuckDB 接続（ファイル DB を作成／既存ファイルを開く）-------------
con = duckdb.connect("data/sensor.duckdb")  # ':memory:' ならメモリ DB

# ---- 2. DataFrame を一時ビューとして登録 -------------------------------
con.register("tmp_df", df)                  # df → DuckDB 内部ビュー tmp_df

# ---- 3-A. 新規テーブルとして保存 (存在しなければ作成) --------------------
con.execute("""
    CREATE TABLE IF NOT EXISTS sensor_data AS
    SELECT * FROM tmp_df
""")

# ---- 3-B. 既存テーブルに追記したい場合 ------------------------------------
# con.execute("INSERT INTO sensor_data SELECT * FROM tmp_df")

# ---- 4. 後片付け ----------------------------------------------------------
con.unregister("tmp_df")  # ビューを外す（省略可）
con.close()
```

---

## 2. Polars LazyFrame / DataFrame を登録する例

DuckDB 0.9 以降なら Polars DataFrame を直接登録できます（内部で Arrow 変換）。

```python
import polars as pl
import duckdb

lf = pl.scan_csv("logs/sensor_20250519.csv", try_parse_dates=True)
df = lf.collect()                            # 実体化（LazyFrame のままでも OK）

con = duckdb.connect("data/sensor.duckdb")

# Polars → DuckDB へ一時ビュー登録
con.register("tmp_pl_df", df)                # → Arrow 経由で取り込まれる

con.execute("""
    CREATE OR REPLACE TABLE sensor_logs AS
    SELECT * FROM tmp_pl_df
""")

con.close()
```

---

### よくある質問 Q&A

|Q|A|
|---|---|
|**既存テーブルに同じ列構成で追記するだけ？**|`INSERT INTO table SELECT * FROM tmp_view` にすれば OK。|
|**型を明示したい**|`CREATE TABLE new_tbl (col1 INT, …); INSERT INTO …` の 2 ステップで。|
|**重複防止・PK は？**|`CREATE TABLE … PRIMARY KEY (…)` でキー列を決め、`ON CONFLICT DO NOTHING` は未サポートなので重複チェック用の一時テーブルを挟むのが定石です。|
|**大量データを高速ロード**|CSV/Parquet なら `COPY table FROM 'file.csv'`、Python 経由なら `con.execute("CREATE TABLE … AS SELECT * FROM read_parquet('file.parquet')")` の方が速いこともあります。|

この 2 ステップパターン（DataFrame → ビュー登録 → SELECT で永続化）を覚えておけば、Pandas と Polars のどちらでもほぼ同じ書き方で DuckDB への登録が行えます。


### 縦持ちのデータフレームを横持ちにする方法

縦持ちデータのカラム
- pram_id
- param_name_en
- param_name_ja
- plant_name
- machine_no
- data_source
- insert_date

横持ちにしたい
headerはマルチカラムインデックス
header = [plant_name, machine_no, data_source]
index = [param_name_en, param_name_ja]
values = param_id


```python
import numpy as np
import pandas as pd

# ── 0. カスタム順序を辞書で定義 ──────────────
custom_order = {           # 好きな順序を 0,1,2… で
    "パラメータA": 0,
    "パラメータB": 1,
    "パラメータX": 2,
}

# wide: すでに pivot 済みの DataFrame
# index = (param_name_en, param_name_ja)

wide_sorted = (
    wide.reset_index()                                  # ⇐ MultiIndex → 列
        .assign(                                        # ── 1. ソートキー列 ──
            _key=lambda d: (
                d["param_name_ja"]
                  .map(custom_order)                    # 指定済み → 0,1,2…
                  .fillna(np.inf)                      # 未指定 → 無限大
            )
        )
        .sort_values(                                   # ── 2. 並べ替え ──
            by=["_key", "param_name_ja", "param_name_en"]
            # _key 昇順 ⇒ 指定ありが先
            # 同 tie なら param_name_ja の昇順
        )
        .drop(columns="_key")                           # ── 3. 後片付け ──
        .set_index(["param_name_en", "param_name_ja"])  # 列 → MultiIndex
)

```


 