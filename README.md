# Приводим необходимые поля к timedelta
for col in ["Отработанное время", "Общая продолжительность",
            "Время отсутствия", "Время обеда"]:
    df[col] = pd.to_timedelta(df[col], errors="coerce")

# Новое поле
df["Новое отработанное время"] = pd.NaT

# ---- УСЛОВИЕ 1 УДАЛЕНО ----
# НЕТ блока "если >= 24 часов"

# Общая маска для всех остальных строк
mask_other = pd.Series([True] * len(df))

# 2. Если время выхода < времени входа → сотрудник вышел на следующий день
mask_next_day = mask_other & (df["Время выхода"] < df["Время входа"])

df.loc[mask_next_day, "Новое отработанное время"] = (
    pd.Timedelta(hours=24)
    - (df.loc[mask_next_day, "Время входа"] - df.loc[mask_next_day, "Время входа"].dt.normalize())
    + (df.loc[mask_next_day, "Время выхода"] - df.loc[mask_next_day, "Время выхода"].dt.normalize())
)

# 3. Обычный расчёт
mask_normal = mask_other & ~mask_next_day
raw_work = df["Время выхода"] - df["Время входа"]
df.loc[mask_normal, "Новое отработанное время"] = raw_work[mask_normal]

# 4. Разделяем <5 часов и ≥5 часов
mask_lt_5 = mask_normal & (raw_work < pd.Timedelta(hours=5))
mask_ge_5 = mask_normal & (raw_work >= pd.Timedelta(hours=5))

# 5. Если < 5 часов → вычитаем только "Время отсутствия"
df.loc[mask_lt_5, "Новое отработанное время"] = (
    raw_work[mask_lt_5] - df.loc[mask_lt_5, "Время отсутствия"]
)

# 6. Если ≥ 5 часов → вычитаем макс(обед, отсутствие)
max_break = df[["Время обеда", "Время отсутствия"]].max(axis=1)
df.loc[mask_ge_5, "Новое отработанное время"] = (
    raw_work[mask_ge_5] - max_break[mask_ge_5]
)
