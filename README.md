import pandas as pd

# Загружаем таблицы
df_main = pd.read_excel("main.xlsx")       # таблица, где ищем совпадения
df_ref = pd.read_excel("reference.xlsx")   # таблица-справочник с M и V

# Приводим сравниваемые колонки к единому формату (важно!)
df_main['N'] = df_main['N'].astype(str).str.strip()
df_ref['M'] = df_ref['M'].astype(str).str.strip()

# Выполняем VLOOKUP (merge по ключу N=M)
df_merged = df_main.merge(
    df_ref[['M', 'V']],        # берем только колонки-ключ и значение
    how='left',                # left — чтобы не потерять строки из основной таблицы
    left_on='N',
    right_on='M'
)

# Записываем результат в новый столбец, например "V_match"
df_merged.rename(columns={'V': 'V_match'}, inplace=True)

# Убираем дублирующий столбец M после merge
df_merged = df_merged.drop(columns=['M'])

# Сохраняем результат
df_merged.to_excel("result.xlsx", index=False)
