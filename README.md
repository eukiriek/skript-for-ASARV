import pandas as pd

# Загружаем файл
df = pd.read_excel("your_file.xlsx")

# Если столбца N нет — создадим его пустым
if 'N' not in df.columns:
    df['N'] = pd.NA

# Базовая маска: где N пустой (NaN или пустая строка)
mask_N_empty = df['N'].isna() | (df['N'].astype(str).str.strip() == '')

# Преобразуем M в "время" через timedelta (формат HH:MM:SS)
time_M = pd.to_timedelta(df['M'].astype(str), errors='coerce')

# --- 1. Если N пусто и "сурв" == 1 → N = 06:00:00
mask1 = mask_N_empty & (df['сурв'] == 1)
df.loc[mask1, 'N'] = '06:00:00'

# Обновляем маску пустых N (после первого шага часть уже заполнилась)
mask_N_empty = df['N'].isna() | (df['N'].astype(str).str.strip() == '')

# --- 2. Если N пусто и время в M == 08:00:00 → N = 01:00:00
mask2 = mask_N_empty & (time_M == pd.Timedelta(hours=8))
df.loc[mask2, 'N'] = '01:00:00'

# Обновляем маску снова
mask_N_empty = df['N'].isna() | (df['N'].astype(str).str.strip() == '')

# --- 3. Если N пусто и время в M между 04:00:00 и 05:00:00 → N = 02:00:00
mask3 = mask_N_empty & (time_M >= pd.Timedelta(hours=4)) & (time_M <= pd.Timedelta(hours=5))
df.loc[mask3, 'N'] = '02:00:00'

# Обновляем маску ещё раз
mask_N_empty = df['N'].isna() | (df['N'].astype(str).str.strip() == '')

# --- 4. Все остальные пустые N → 04:00:00
# Если нужно именно число 4, то поменяй строку ниже на: df.loc[mask_N_empty, 'N'] = 4
df.loc[mask_N_empty, 'N'] = '04:00:00'

# Сохраняем результат
df.to_excel("your_file_updated.xlsx", index=False)

print("Готово! Столбец N заполнен по заданным условиям.")
