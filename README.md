import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем M в datetime (важно!)
df['M'] = pd.to_datetime(df['M'], errors='coerce')

# Нормализуем текст в столбце N (убираем пробелы, приводим к нижнему регистру)
df['N_clean'] = df['N'].astype(str).str.strip().str.lower()

# Список значений для ПН–ЧТ (поддерживаем полные и краткие формы)
weekdays_mon_to_thu = [
    'понедельник', 'вторник', 'среда', 'четверг',
    'пн', 'вт', 'ср', 'чт'
]

# Маска на строки, где день недели попадает в ПН–ЧТ
mask = df['N_clean'].isin(weekdays_mon_to_thu)

# Меняем только время, дата остаётся прежней
df.loc[mask, 'M'] = df.loc[mask, 'M'].dt.normalize() + pd.Timedelta(hours=8, minutes=15)

# Приводим к формату dd.mm.yyyy HH:MM:SS
df['M'] = df['M'].dt.strftime('%d.%m.%Y %H:%M:%S')

# Удаляем вспомогательный столбец
df = df.drop(columns=['N_clean'])

df.to_excel("your_file_updated.xlsx", index=False)
