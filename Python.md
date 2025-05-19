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
