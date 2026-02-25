
import pandas as pd
import numpy as np

# Читаем файл
df = pd.read_excel("your_file.xlsx")

# Берем первый столбец (по индексу 0)
col = df.iloc[:, 0]

# Приводим к строке, убираем пробелы и переносы
clean_col = (
    col.astype(str)
       .replace(r'^\s*$', np.nan, regex=True)   # строки из пробелов и \n → NaN
)

# Удаляем строки, где первый столбец пустой
df_cleaned = df[clean_col.notna()].copy()

# Сохраняем результат
df_cleaned.to_excel("cleaned_file.xlsx", index=False)

print("Пустые строки удалены.")
