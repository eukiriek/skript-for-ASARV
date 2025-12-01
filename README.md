df["Новое отработанное время"] = pd.NaT

# ===== Условие 1 =====
# Если отработанное время >= 24 часов → просто копируем
mask_ge_24 = df["Отработанное время"] >= pd.Timedelta(hours=24)
df.loc[mask_ge_24, "Новое отработанное время"] = df.loc[mask_ge_24, "Отработанное время"]

# Общая маска для остальных
mask_other = ~mask_ge_24


# ===== Условие 2 =====
# Если время выхода < времени входа → сотрудник вышел на следующий день
mask_next_day = mask_other & (df["Время выхода"] < df["Время входа"])

# Расчёт по правилу:
# новое_время = (24 часа - время входа) + время выхода
df.loc[mask_next_day, "Новое отработанное время"] = (
    pd.Timedelta(hours=24) - (df.loc[mask_next_day, "Время входа"] - df.loc[mask_next_day, "Время входа"].dt.normalize())
    + (df.loc[mask_next_day, "Время выхода"] - df.loc[mask_next_day, "Время выхода"].dt.normalize())
)


# ===== Условие 3 =====
# Оставшиеся строки → обычный расчёт как выход - вход
mask_normal = mask_other & ~mask_next_day

raw_work = df["Время выхода"] - df["Время входа"]
df.loc[mask_normal, "Новое отработанное время"] = raw_work[mask_normal]

# Разделяем на <5 часов и >=5 часов
mask_lt_5 = mask_normal & (raw_work < pd.Timedelta(hours=5))
mask_ge_5 = mask_normal & (raw_work >= pd.Timedelta(hours=5))

# Если < 5 часов → вычитаем только "Время отсутствия"
df.loc[mask_lt_5, "Новое отработанное время"] = (
    raw_work[mask_lt_5] - df.loc[mask_lt_5, "Время отсутствия"]
)

# Если ≥ 5 часов → вычитаем max(обед, отсутствие)
max_break = df[["Время обеда", "Время отсутствия"]].max(axis=1)
df.loc[mask_ge_5, "Новое отработанное время"] = (
    raw_work[mask_ge_5] - max_break[mask_ge_5]
)
