import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим колонку N к безопасному строковому формату
df["N"] = df["N"].astype(str)

# Создаём новый столбец M по условию
df["M"] = df["N"].str.contains("значение", case=False, na=False).astype(int)

df.to_excel("your_file_updated.xlsx", index=False)

print("Готово! Столбец M создан.")
