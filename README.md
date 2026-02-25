import pandas as pd
import numpy as np

# Читаем файл
df = pd.read_excel("your_file.xlsx")

# Работаем ТОЛЬКО со столбцом "Поле 1"
df["Поле 1"] = (
    df["Поле 1"]
        .astype(str)
        .replace(r'^\s*$', np.nan, regex=True)  # пусто, пробелы, перенос строки → NaN
)

# Удаляем строки, где "Поле 1" пустой
df_cleaned = df.dropna(subset=["Поле 1"])

# Сохраняем результат
df_cleaned.to_excel("cleaned_file.xlsx", index=False)

print("Строки, где 'Поле 1' пустое или содержит только перенос строки, удалены.")
