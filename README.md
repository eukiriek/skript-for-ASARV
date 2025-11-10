import pandas as pd
from datetime import time

# Загружаем файл
df = pd.read_excel("your_file.xlsx")

# Преобразуем M в datetime (дата + время если было)
df['M'] = pd.to_datetime(df['M'], format='%d.%m.%Y', errors='coerce')

# Получаем номер дня недели: Monday=0 ... Sunday=6
weekday = df['M'].dt.weekday

# Условие: Пн(0), Вт(1), Ср(2), Чт(3)
mask = weekday.isin([0, 1, 2, 3])

# Меняем время на 08:15:00
df.loc[mask, 'M'] = df.loc[mask, 'M'].dt.normalize() + pd.Timedelta(hours=8, minutes=15)

# Сохраняем обратно
df.to_excel("your_file_updated.xlsx", index=False)
