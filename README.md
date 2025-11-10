import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Название столбца, который нужно преобразовать
col = "Time"

# 1) Приводим к строке и очищаем пробелы
df[col] = df[col].astype(str).str.strip()

# 2) Преобразуем к формату времени
df[col] = pd.to_datetime(df[col], errors='coerce').dt.time

# 3) Если нужно формат именно как строка "ЧЧ:ММ:СС":
df[col] = df[col].astype(str)

# Сохраняем
df.to_excel("your_file_time_formatted.xlsx", index=False)


import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим значения в M к формату времени ЧЧ:ММ:СС
df['M'] = pd.to_datetime(df['M'], errors='coerce').dt.strftime('%H:%M:%S')

df.to_excel("your_file_time_formatted.xlsx", index=False)


import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим колонку M к формату времени
df['M'] = pd.to_datetime(df['M'], errors='coerce').dt.strftime('%H:%M:%S')

# Удаляем строки, где время отсутствует или равно 00:00:00
df = df[ df['M'].notna() & (df['M'] != '00:00:00') ]

df.to_excel("your_file_time_filtered.xlsx", index=False)

