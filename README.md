import pandas as pd

# Загружаем файл
df = pd.read_excel("your_file.xlsx")

# Названия колонок — подставь свои, если отличаются
col_worked = "отработанное время"
col_out = "время выхода"
col_in = "время входа"
col_1 = "поле 1"
col_2 = "поле 2"
col_new = "новое отработанное время"

# Преобразуем все нужные колонки в Timedelta (формат чч:мм:сс)
for col in [col_worked, col_out, col_in, col_1, col_2]:
    df[col] = pd.to_timedelta(df[col].astype(str), errors="coerce")

# Условие: отработанное время > 25 часов
mask_big = df[col_worked] > pd.Timedelta(hours=25)

# Создаём новый столбец
df[col_new] = pd.NaT

# 1) Если отработанное время > 25 часов → оставляем как есть
df.loc[mask_big, col_new] = df.loc[mask_big, col_worked]

# 2) Иначе считаем: время выхода − время входа − максимум из (поле 1, поле 2)
max_break = df[[col_1, col_2]].max(axis=1)

df.loc[~mask_big, col_new] = (
    df.loc[~mask_big, col_out]
    - df.loc[~mask_big, col_in]
    - max_break[~mask_big]
)

# (опционально) сохранить обратно в Excel
df.to_excel("your_file_updated.xlsx", index=False)
