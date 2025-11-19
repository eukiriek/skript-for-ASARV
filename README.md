# строковый формат времени
norma_str = df['Норма'].astype(str).str.strip()

# вытаскиваем часы, минуты и секунды из формата H:MM:SS или HH:MM:SS
time_parts = norma_str.str.extract(r'(\d+):(\d{1,2}):(\d{1,2})')

# заменяем NaN на 0 и приводим к int
time_parts = time_parts.fillna(0).astype(int)

hours   = time_parts[0]
minutes = time_parts[1]
seconds = time_parts[2]

total_seconds = hours * 3600 + minutes * 60 + seconds

# флаг: Норма < 8 часов (8 часов НЕ включаем)
df['Флаг норма<8'] = (total_seconds < 8 * 3600).astype(int)
