# Новое поле
df["Новое отработанное время"] = pd.NaT

# ===== 1. Сначала обрабатываем строки с СУРВ = 1 =====
mask_surv = df["СУРВ"] == 1

# Норма / Неотработанное / Переработанное приводим к timedelta
norm_surv = pd.to_timedelta(df.loc[mask_surv, "Норма"], errors="coerce")

not_work_surv = pd.to_timedelta(
    df.loc[mask_surv, "Неотработанное время"], errors="coerce"
).fillna(pd.Timedelta(0))

over_surv = pd.to_timedelta(
    df.loc[mask_surv, "Переработанное время"], errors="coerce"
).fillna(pd.Timedelta(0))

# Формула для СУРВ = 1:
# Новое отработанное время = Норма - Неотработанное + Переработанное
df.loc[mask_surv, "Новое отработанное время"] = norm_surv - not_work_surv + over_surv

# ===== 2. Все остальные строки — по старой логике =====
mask_other = ~mask_surv

# Если время выхода < времени входа → сотрудник вышел на следующий день
mask_next_day = mask_other & (df["Время выхода"] < df["Время входа"])

df.loc[mask_next_day, "Новое отработанное время"] = (
    pd.Timedelta(hours=24)
    - (df.loc[mask_next_day, "Время входа"] - df.loc[mask_next_day, "Время входа"].dt.normalize())
    + (df.loc[mask_next_day, "Время выхода"] - df.loc[mask_next_day, "Время выхода"].dt.normalize())
)

# Обычный расчёт
mask_normal = mask_other & ~mask_next_day
raw_work = df["Время выхода"] - df["Время входа"]
df.loc[mask_normal, "Новое отработанное время"] = raw_work[mask_normal]

# Разделяем < 5 часов и ≥ 5 часов (только для обычных строк)
mask_lt_5 = mask_normal & (raw_work < pd.Timedelta(hours=5))
mask_ge_5 = mask_normal & (raw_work >= pd.Timedelta(hours=5))

# Если < 5 часов → вычитаем только "Время отсутствие итог"
df.loc[mask_lt_5, "Новое отработанное время"] = (
    raw_work[mask_lt_5] - df.loc[mask_lt_5, "Время отсутствие итог"]
)

# Если ≥ 5 часов → вычитаем max(обед, отсутствие)
max_break = df[["Время обеда", "Время отсутствие итог"]].max(axis=1)
df.loc[mask_ge_5, "Новое отработанное время"] = (
    raw_work[mask_ge_5] - max_break[mask_ge_5]
)

print('-- расчет поля Новое отработанное время.')
