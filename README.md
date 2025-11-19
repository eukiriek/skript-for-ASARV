import pandas as pd

# приводим 'Норма' к строке
norma_str = df['Норма'].astype(str).str.strip()

# разбираем время формата H:MM:SS или HH:MM:SS (в т.ч. 0:30:00, 07:00:00 и т.п.)
time_parts = norma_str.str.extract(
    r'^\s*(\d{1,2}):(\d{1,2}):(\d{1,2})\s*$'
)

# NaN → 0 и в int
time_parts = time_parts.fillna(0).astype(int)

hours   = time_parts[0]
minutes = time_parts[1]
seconds = time_parts[2]

# считаем общее количество секунд
total_seconds = hours * 3600 + minutes * 60 + seconds

# флаг: Норма < 8 часов (8 часов НЕ включаем)
df['Флаг норма<8'] = (total_seconds < 8 * 3600).astype(int)
