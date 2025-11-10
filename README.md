import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем M в datetime (если в ней дата или дата+время — это нормально)
df['M'] = pd.to_datetime(df['M'], errors='coerce')

# Нормализуем текст в столбце N
df['N_clean'] = df['N'].astype(str).str.strip().str.lower()

# Значения для Пн–Чт
weekdays_mon_to_thu = [
    'понедельник', 'вторник', 'среда', 'четверг',
    'пн', 'вт', 'ср', 'чт'
]

# Преобразуем L в числовой вид
df['L'] = pd.to_numeric(df['L'], errors='coerce')

# Маска: день недели Пн–Чт И при этом L == 0
mask = (df['N_clean'].isin(weekdays_mon_to_thu)) & (df['L'] == 0)

# Меняем время только для этих строк
df.loc[mask, 'M'] = df.loc[mask, 'M'].dt.normalize() + pd.Timedelta(hours=8, minutes=15)

# Приводим M к строковому формату только времени
df['M'] = df['M'].dt.strftime('%H:%M:%S')

# Убираем технический столбец
df = df.drop(columns=['N_clean'])

df.to_excel("your_file_updated.xlsx", index=False)
