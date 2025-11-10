import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем столбец M в datetime (если там только дата — это нормально)
df['M'] = pd.to_datetime(df['M'], errors='coerce')

# Список значений дней недели, при которых нужно менять время
weekdays_target = ['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пн', 'Вт', 'Ср', 'Чт']

# Создаем логическую маску
mask = df['N'].isin(weekdays_target)

# Меняем время на 08:15:00 (сохраняя дату)
df.loc[mask, 'M'] = df.loc[mask, 'M'].dt.normalize() + pd.Timedelta(hours=8, minutes=15)

# Если нужно сохранить в формате строки dd.mm.yyyy HH:MM:SS:
df['M'] = df['M'].dt.strftime('%d.%m.%Y %H:%M:%S')

df.to_excel("your_file_updated.xlsx", index=False)
