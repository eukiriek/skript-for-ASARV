import pandas as pd

# Загружаем таблицы
df_main = pd.read_excel("main.xlsx")       # основная таблица
df_ref = pd.read_excel("reference.xlsx")   # справочник

# Приводим сравниваемые колонки к строковому формату без пробелов
df_main['N'] = df_main['N'].astype(str).str.strip()
df_ref['M'] = df_ref['M'].astype(str).str.strip()

# Определяем значения, которые считаются совпадениями
match_values = set(df_ref['M'].unique())

# Фильтруем строки: оставляем только те, где значения из N нет в списке M
df_filtered = df_main[~df_main['N'].isin(match_values)].copy()

# Сохраняем результат
df_filtered.to_excel("result.xlsx", index=False)

print(f"Удалено строк: {len(df_main) - len(df_filtered)}")
print("Файл result.xlsx успешно создан.")
