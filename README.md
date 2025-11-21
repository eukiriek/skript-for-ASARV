import pandas as pd

# Загружаем исходный файл
df = pd.read_excel("input.xlsx")

# 1. Приводим время к timedelta
df["время"] = pd.to_timedelta(df["время"])

# 2. Группировка + сумма
agg = df.groupby("сотрудник", as_index=False)["время"].sum()

# 3. Сохраняем результат в новый лист Excel
with pd.ExcelWriter("output.xlsx") as writer:
    df.to_excel(writer, sheet_name="Исходные_данные", index=False)
    agg.to_excel(writer, sheet_name="Агрегировано", index=False)
