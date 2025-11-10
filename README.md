import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим значения в Z к строке и убираем пробелы
df['Z'] = df['Z'].astype(str).str.strip()

# Вариант 1: если в таблице время как "00:00:00"
df = df[df['Z'] != '00:00:00']

# Вариант 2: если в таблице время как "00.00.00" (поменяйте строку ниже)
# df = df[df['Z'] != '00.00.00']

df.to_excel("your_file_updated.xlsx", index=False)


import pandas as pd

# 1) Читаем файл
df = pd.read_excel("your_file.xlsx")

# 2) Нормализуем текст в столбце N (дни недели заданы текстом)
df['N_clean'] = df['N'].astype(str).str.strip().str.lower()

# 3) Целевые дни недели (Пн–Чт)
weekdays_mon_to_thu = [
    'понедельник', 'вторник', 'среда', 'четверг',
    'пн', 'вт', 'ср', 'чт'
]

mask_mon_thu = df['N_clean'].isin(weekdays_mon_to_thu)

# 4) Приводим M к datetime (может быть время или дата+время; ошибки -> NaT)
m_dt = pd.to_datetime(df['M'], errors='coerce')

# 5) Маска "время в M < 08:00:00" (учитываем только распознанные времена)
secs = (m_dt.dt.hour.fillna(0).astype(int) * 3600 +
        m_dt.dt.minute.fillna(0).astype(int) * 60 +
        m_dt.dt.second.fillna(0).astype(int))
mask_time_lt_8 = m_dt.notna() & (secs < 8 * 3600)

# 6) Маска L == 1 (тогда ничего не делаем)
mask_L_eq_1 = (df['L'] == 1)

# 7) Итоговая маска применения правила:
#    применяем ТОЛЬКО если день Пн–Чт, L != 1, и время не меньше 08:00
mask_apply = mask_mon_thu & (~mask_L_eq_1) & (~mask_time_lt_8)

# 8) Обновляем M на 08:15:00 там, где mask_apply
df.loc[mask_apply, 'M'] = '08:15:00'

# 9) Приводим M строго к формату времени HH:MM:SS (без даты)
m_dt2 = pd.to_datetime(df['M'], errors='coerce')
df['M'] = m_dt2.dt.strftime('%H:%M:%S')
df.loc[m_dt2.isna(), 'M'] = None  # если не распознано — оставим пусто

# 10) Чистим тех. столбец и сохраняем
df = df.drop(columns=['N_clean'])
df.to_excel("your_file_updated.xlsx", index=False)



import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Приводим M к datetime (если там только время — дата подставится автоматически)
m_dt = pd.to_datetime(df['M'], errors='coerce')

# Считаем время в секундах с начала суток
seconds = (
    m_dt.dt.hour.fillna(0).astype(int) * 3600 +
    m_dt.dt.minute.fillna(0).astype(int) * 60 +
    m_dt.dt.second.fillna(0).astype(int)
)

# Создаем новый столбец-признак
# 1 — если время < 08:00, иначе 0
df['Time_lt_8'] = (seconds < 8 * 3600).astype(int)

df.to_excel("your_file_updated.xlsx", index=False)
