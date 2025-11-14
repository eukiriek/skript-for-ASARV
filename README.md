import pandas as pd
import numpy as np

# Загружаем файл
df = pd.read_excel("your_file.xlsx")

# --- Преобразование M и K в формат времени ---
df["M"] = pd.to_timedelta(df["M"].astype(str), errors="coerce")
df["K"] = pd.to_timedelta(df["K"].astype(str), errors="coerce")

# --- Создаём новое поле N ---
# Вычисляем разницу
df["N"] = df["M"] - df["K"]

# Если разница <= 0 или NaT → ставим 0
df.loc[(df["N"].dt.total_seconds() <= 0) | (df["N"].isna()), "N"] = pd.Timedelta(0)

# --- Преобразуем N обратно в формат чч:мм:сс ---
df["N"] = df["N"].dt.strftime("%H:%M:%S")

# Сохраняем результат
df.to_excel("updated.xlsx", index=False)

print("Готово! Поле N успешно создано.")
