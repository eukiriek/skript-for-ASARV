import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим колонку N к безопасному строковому формату
df["N"] = df["N"].astype(str)

# Создаём новый столбец M по условию
df["M"] = df["N"].str.contains("значение", case=False, na=False).astype(int)

df.to_excel("your_file_updated.xlsx", index=False)

print("Готово! Столбец M создан.")



import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим колонки к строковому формату
df["N"] = df["N"].astype(str)
df["F"] = df["F"].astype(str)
df["G"] = df["G"].astype(str)

# Условие 1: колонка N содержит слово "значение"
cond1 = df["N"].str.contains("значение", case=False, na=False)

# Условие 2: F содержит "значение2" И G содержит "значение3"
cond2 = (
    df["F"].str.contains("значение2", case=False, na=False) &
    df["G"].str.contains("значение3", case=False, na=False)
)

# Если выполняется хотя бы одно условие → 1, иначе → 0
df["M"] = ((cond1) | (cond2)).astype(int)

df.to_excel("your_file_updated.xlsx", index=False)

print("Готово! Столбец M создан и заполнен по двум условиям.")
