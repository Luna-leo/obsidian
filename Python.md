

### pandas の場合（`param_name` を行インデックスにせず、普通の列にする）

```python
df_wide = (
    df.pivot_table(
        index="param_name",
        columns=["plant_name", "unit", "data_source"],
        values="param_id",
        aggfunc="first"
    )
    .reset_index()          # ← ここでインデックスを通常の列に戻す
    .rename_axis(None, axis=1)  # 列名のインデックス名も消す（お好みで）
)

# param_name が不要なら → .reset_index(drop=True)
```

- `reset_index()` …… 行インデックスだった **`param_name`** が普通の列に移動し、  
              デフォルトの整数インデックス（0,1,2 …）が付く
    
- `rename_axis(None, axis=1)` …… 列ヘッダー上部の “plant_name | unit | data_source” という  
                  名前（`columns.name`）を消したいときだけ付けてください
    

---

### Polars の場合

Polars にはそもそも行インデックスの概念がないので、上の `pivot()` を実行した時点で  
`param_name` は **ただの列** になっています。追加の操作は不要です。

```python
wide = (
    pl.from_pandas(df)
      .pivot(
          index="param_name",
          columns=["plant_name", "unit", "data_source"],
          values="param_id",
          aggregate_function="first"
      )
)
# そのまま wide は行番号 0,1,2… の DataFrame
```

---

#### まとめ

- **pandas** では `pivot_table` → **`reset_index()`** で「インデックスなし」にできる
    
- **Polars** ではインデックスが無いのでそのまま
    

これで `param_name` が列として残り、行はデフォルトの整数インデックスになります。