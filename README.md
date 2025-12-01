# ==============================
# добавление поля новое время прихода/выхода
# ==============================

# Шаг 1. Нормализуем исходные значения как timedelta
vhod_raw = df["Время входа"].astype(str).str.strip().str.replace('.', ':', regex=False)
vyhod_raw = df["Время выхода"].astype(str).str.strip().str.replace('.', ':', regex=False)

t_vhod = pd.to_timedelta(vhod_raw, errors="coerce")
t_vyhod = pd.to_timedelta(vyhod_raw, errors="coerce")

# Шаг 2. Если выход меньше входа — считаем, что выход был на следующий день (+24 часа)
mask_next_day = t_vyhod < t_vhod
t_vyhod[mask_next_day] = t_vyhod[mask_next_day] + pd.Timedelta(days=1)

# Шаг 3. Новое время входа/выхода: +30 минут и округление до часа вниз
t_new_vhod = (t_vhod + pd.Timedelta(minutes=30)).dt.floor("H")
t_new_vyhod = (t_vyhod + pd.Timedelta(minutes=30)).dt.floor("H")

# Шаг 4. Переводим обратно в строки HH:MM:SS
df["Время входа"] = t_vhod.apply(td_to_hms)
df["Время выхода"] = t_vyhod.apply(td_to_hms)
df["Новое время входа"] = t_new_vhod.apply(td_to_hms)
df["Новое время выхода"] = t_new_vyhod.apply(td_to_hms)

print("-- добавление поля новое время прихода, новое время выхода")
