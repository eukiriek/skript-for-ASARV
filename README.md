import pandas as pd

col_base = "Время отсутствия"
col_sub  = "Время отсутствие из enters"    # поправь название, если у тебя чуть другое
col_res  = "Время отсутствие финальное"

# 1. Приводим к строкам и чистим пробелы
df[col_base] = df[col_base].astype(str).str.strip()
df[col_sub]  = df[col_sub].astype(str).str.strip()

# пустые строки считаем как отсутствующие
df[col_base] = df[col_base].replace({"": pd.NA})
df[col_sub]  = df[col_sub].replace({"": pd.NA})

# 2. Переводим в timedelta
t_base = pd.to_timedelta(df[col_base], errors="coerce")
t_sub  = pd.to_timedelta(df[col_sub],  errors="coerce")

# 3. Условия, когда НИЧЕГО НЕ ДЕЛАТЬ (оставить как есть):
#   - базовое время пустое или равно 0
#   - или базовое время < времени из enters
zero_or_empty = t_base.isna() | (t_base == pd.Timedelta(0))
less_than_sub = t_base < t_sub

# где можно вычитать: базовое не пустое/не ноль, не меньше, и есть что вычитать
mask_subtract = ~(zero_or_empty | less_than_sub | t_sub.isna())

# 4. Формируем финальное поле (по умолчанию = исходное время)
t_final = t_base.copy()

# там, где можно вычитать — вычитаем
t_final[mask_subtract] = t_base[mask_subtract] - t_sub[mask_subtract]

# все NaT (пустые) считаем нулём
t_final = t_final.fillna(pd.Timedelta(0))

# 5. Переводим обратно в строку формата 00:00:00
df[col_res] = t_final.dt.components.apply(
    lambda r: f"{int(r['hours']):02d}:{int(r['minutes']):02d}:{int(r['seconds']):02d}",
    axis=1
)
