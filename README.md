import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем M в формат даты (если там дата+время — выделим дату)
df['M'] = pd.to_datetime(df['M'], errors='coerce')

# Создаем новый столбец "DayOfWeek" с названием дня недели
df['DayOfWeek'] = df['M'].dt.day_name()  # названия на английском

# Если нужен русский язык:
# df['DayOfWeek'] = df['M'].dt.day_name(locale='ru_RU')

df.to_excel("your_file_with_dayofweek.xlsx", index=False)

df['DayOfWeek'] = df['M'].dt.strftime('%a')

