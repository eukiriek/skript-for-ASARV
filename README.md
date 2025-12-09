import pandas as pd
import numpy as np

# ===== 1. Чтение файла =====
file_path = "data.xlsx"   # путь к вашему файлу
sheet_name = 0            # или имя листа, где лежат исходные данные

df = pd.read_excel(file_path, sheet_name=sheet_name)

# ===== 2. Приведение типов =====
# Названия колонок ориентировочные — поправьте под свои, если отличаются
df["Начало"] = pd.to_datetime(df["Начало"], errors="coerce", dayfirst=True)

# "Время" в формате ч:мм:сс / чч:мм:сс → timedelta
df["Время"] = pd.to_timedelta(df["Время"].astype(str), errors="coerce")

# Оставим только нужные поля (не обязательно, но удобнее)
df = df[["Имя", "Начало", "Вид временного события", "Время"]].copy()

# Убираем строки без имени, даты или времени
df = df.dropna(subset=["Имя", "Начало", "Время"])

# ===== 3. Сортировка по человеку/дню/времени =====
df = df.sort_values(["Имя", "Начало", "Время"])

# ===== 4. Сдвиги внутри группы (Имя + Начало) =====
df["Время_след"] = df.groupby(["Имя", "Начало"])["Время"].shift(-1)
df["Событие_след"] = df.groupby(["Имя", "Начало"])["Вид временного события"].shift(-1)

# ===== 5. Расчет интервалов "Приход–Уход" =====
mask_prihod = df["Вид временного события"].str.lower().str.strip().eq("приход")
mask_next_uho d = df["Событие_след"].str.lower().str.strip().eq("уход")

# интервал только там, где сейчас Приход, а следующее событие — Уход
df["Интервал"] = np.where(
    mask_prihod & mask_next_uhod,
    df["Время_след"] - df["Время"],
    pd.Timedelta(0)
)

# Если есть отрицательные/битые интервалы – на всякий случай обнулим
df.loc[df["Интервал"] < pd.Timedelta(0), "Интервал"] = pd.Timedelta(0)

# ===== 6. Агрегация по человеку и дню =====
agg = (
    df.groupby(["Имя", "Начало"], as_index=False)["Интервал"]
      .sum()
)

# Дополнительно можно сделать строковое представление времени
agg["Интервал_чч:мм:сс"] = agg["Интервал"].astype("timedelta64[s]").astype(int)
agg["Интервал_чч:мм:сс"] = pd.to_timedelta(agg["Интервал_чч:мм:сс"], unit="s")

# ===== 7. Запись в новый лист того же файла =====
from openpyxl import load_workbook

book = load_workbook(file_path)
with pd.ExcelWriter(file_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as writer:
    writer.book = book
    agg.to_excel(writer, sheet_name="Свод_по_дням", index=False)
