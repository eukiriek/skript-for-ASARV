df["Отсутствие"] = pd.Timedelta(0)
df.loc[mask_gap, "Отсутствие"] = df.loc[mask_gap, "Время"] - df.loc[mask_gap, "prev_time"]
df.loc[df["Отсутствие"] < pd.Timedelta(0), "Отсутствие"] = pd.Timedelta(0)

# --- НОВЫЙ БЛОК: оставляем только промежутки 5–15 минут ---
min_gap = pd.Timedelta(minutes=5)
max_gap = pd.Timedelta(minutes=15)

mask_len = (df["Отсутствие"] >= min_gap) & (df["Отсутствие"] <= max_gap)
# всё, что меньше 5 минут или больше 15 минут — не участвует в сумме
df.loc[~mask_len, "Отсутствие"] = pd.Timedelta(0)
# --- КОНЕЦ НОВОГО БЛОКА ---
