import pandas as pd

col_base = "Время отсутствия"
col_sub  = "Время отсутствие из enters"    # имя столбца с enters
col_res  = "Время отсутствие финальное"

# 1. Чистим строки
df[col_base] = df[col_base].astype(str).str.strip()
df[col_sub]  = df[col_sub].astype(str).str.strip()

df[col_base] = df[col_base].replace({"": pd.NA})
df[col_sub]  = df[col_sub].replace({"": pd.NA})

# 2. В timedelta
t_base = pd.to_timedelta(df[col_base], errors="coerce")
t_sub  = pd.to_timedelta(df[col_sub],  errors="coerce")

# 3. Условия
zero_or_empty = t_base.isna() | (t_base == pd.Timedelta(0))
less_than_sub = t_base < t_sub
can_subtract  = (~zero_or_empty) & (~t_sub.isna()) & (~less_than_sub)

# 4. Финальное время: начинаем с исходного
t_final = t_base.copy()

# 4.1. Если базовое < из enters → ставим время из enters
t_final[less_than_sub & ~zero_or_empty & ~t_sub.isna()] = t_sub[less_than_sub & ~zero_or_empty & ~t_sub.isna()]

# 4.2. Если базовое ≥ из enters и не ноль → вычитаем
t_final[can_subtract] = t_base[can_subtract] - t_sub[can_subtract]

# пустые в ноль, если нужно
t_final = t_final.fillna(pd.Timedelta(0))

# 5. Обратно в строку 00:00:00
df[col_res] = t_final.dt.components.apply(
    lambda r: f"{int(r['hours']):02d}:{int(r['minutes']):02d}:{int(r['seconds']):02d}",
    axis=1
)
