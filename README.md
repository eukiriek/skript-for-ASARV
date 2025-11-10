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

# Маска — где день недели входит в Пн–Чт
mask = df['N_clean'].isin(weekdays_mon_to_thu)

# Меняем время на 08:15:00 (сохранив дату для корректной обработки)
df.loc[mask, 'M'] = df.loc[mask, 'M'].dt.normalize() + pd.Timedelta(hours=8, minutes=15)

# Теперь извлекаем только время в формате HH:MM:SS
df['M'] = df['M'].dt.strftime('%H:%M:%S')

# Убираем технический столбец
df = df.drop(columns=['N_clean'])

df.to_excel("your_file_updated.xlsx", index=False)
