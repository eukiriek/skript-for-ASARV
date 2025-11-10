import pandas as pd

# читаем файл
df = pd.read_excel("your_file.xlsx")

# Нормализуем столбец N к времени:
# 1) приводим к строке, убираем пробелы
# 2) меняем точки на двоеточия (на случай "00.00.00")
# 3) парсим как время HH:MM:SS; непарсящиеся значения -> NaT
n_time = pd.to_datetime(
    df["N"].astype(str).str.strip().str.replace('.', ':', regex=False),
    format="%H:%M:%S",
    errors="coerce"
)

# Маска строк, где время ровно 00:00:00
mask_midnight = (n_time.dt.hour == 0) & (n_time.dt.minute == 0) & (n_time.dt.second == 0)

# Удаляем такие строки
df = df.loc[~mask_midnight].copy()

# Сохраняем результат
df.to_excel("your_file_updated.xlsx", index=False)
