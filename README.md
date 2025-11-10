import pandas as pd

# Основная таблица
df = pd.read_excel("main.xlsx")

# Таблица со списком значений
df_ref = pd.read_excel("reference.xlsx")

# Предположим, что сравниваем значения из столбца M с колонкой Value во второй таблице
# Приведём к строкам/числам, чтобы избежать случайных несовпадений
df['M'] = df['M'].astype(str).str.strip()
df_ref['Value'] = df_ref['Value'].astype(str).str.strip()

# Создаем новую колонку "Match" (как результат ВПР)
df['Match'] = df['M'].isin(df_ref['Value']).astype(int)

df.to_excel("main_with_match.xlsx", index=False)
