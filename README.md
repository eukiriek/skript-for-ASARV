import pandas as pd

# Пример чтения файла
df = pd.read_excel("file.xlsx")

# Убираем переносы строки во всех строковых столбцах
df = df.apply(
    lambda col: col.str.replace(r'\r?\n', ' ', regex=True) 
    if col.dtype == "object" else col
)

# Сохраняем обратно
df.to_excel("file_clean.xlsx", index=False)
