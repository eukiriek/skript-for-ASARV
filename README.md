# --- очистка переносов строк без превращения NaN в "nan"
obj_cols = df.select_dtypes(include=['object']).columns

for col in obj_cols:
    s = df[col]

    # чистим только там, где значение не NaN
    mask = s.notna()
    s2 = (
        s[mask]
        .str.replace('\n', ' ', regex=False)
        .str.replace('\r', ' ', regex=False)
        .str.strip()
    )
    df.loc[mask, col] = s2

# --- приводим "псевдопустые" значения к NA
df[obj_cols] = df[obj_cols].replace(
    {'': pd.NA, ' ': pd.NA, '-': pd.NA, 'nan': pd.NA, 'NaN': pd.NA, 'None': pd.NA, 'none': pd.NA}
)

# --- удаляем строки, которые после чистки полностью пустые
df = df.dropna(how='all')

print('--- переносы строк очищены; пустые строки удалены.')
