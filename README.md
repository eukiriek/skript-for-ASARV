import pandas as pd
import numpy as np

# 1. Загружаем файл
df = pd.read_excel("your_file.xlsx")

# ===== 2. Приводим столбцы к правильным типам =====
# Время входа / выхода обычно дата+время → переводим в datetime
df["время входа"]  = pd.to_datetime(df["время входа"],  errors="coerce")
df["время выхода"] = pd.to_datetime(df["время выхода"], errors="coerce")

# Остальные — это длительности (часы:минуты:секунды) → в Timedelta
for col in ["отработанное время", "время обеда", "время отсутствие"]:
    df[col] = pd.to_timedelta(df[col].astype(str), errors="coerce")

# 3. Создаём новый столбец "новое отработанное время"
df["новое отработанное время"] = pd.NaT    # потом станет Timedelta

# Условие 1: если отработанное время >= 24 часов, просто копируем
mask_ge_24 = df["отработанное время"] >= pd.Timedelta(hours=24)
df.loc[mask_ge_24, "новое отработанное время"] = df.loc[mask_ge_24, "отработанное время"]

# Условие 2: иначе считаем:
# новое = время выхода - время входа - max(время обеда, время отсутствие)
mask_other = ~mask_ge_24

# Максимум из "обеда" и "отсутствия" построчно
max_break = df[["время обеда", "время отсутствие"]].max(axis=1)

df.loc[mask_other, "новое отработанное время"] = (
    df.loc[mask_other, "время выхода"] - df.loc[mask_other, "время входа"] - max_break[mask_other]
)

# ===== 4. Переводим всё в формат ЧЧ:ММ:СС =====

def timedelta_to_hms(td):
    if pd.isna(td):
        return ""
    total_seconds = int(td.total_seconds())
    h = total_seconds // 3600
    m = (total_seconds % 3600) // 60
    s = total_seconds % 60
    return f"{h:02d}:{m:02d}:{s:02d}"

# для длительностей
for col in ["отработанное время", "время обеда", "время отсутствие", "новое отработанное время"]:
    df[col] = df[col].apply(timedelta_to_hms)

# для времени входа/выхода берём только время из datetime
for col in ["время входа", "время выхода"]:
    df[col] = df[col].dt.strftime("%H:%M:%S")

# 5. Сохраняем результат
df.to_excel("your_file_updated.xlsx", index=False)

print("Готово: новое отработанное время рассчитано и все поля приведены к формату ЧЧ:ММ:СС.")
