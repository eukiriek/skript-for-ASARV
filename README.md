import pandas as 
# ========= 5. 

import pandas as pd
import numpy as np
from openpyxl import load_workbook

# ==== Имя файла в текущей папке ====
excel_file = "data.xlsx"     # ← ИЗМЕНИ, если файл называется иначе

# ==== Имя листов ====
source_sheet = "Офисы"       # ← лист с исходными данными
result_sheet = "Время_отсутствия"   # ← лист для результата (и ОН определён!)

# ==== 1. Чтение файла ====
df = pd.read_excel(excel_file, sheet_name=source_sheet)

# ==== 2. Приведение типов ====
df["Имя"] = df["Имя"].astype(str).str.strip()

df["Начало"] = pd.to_datetime(df["Начало"], errors="coerce", dayfirst=True)
df["Начало"] = df["Начало"].dt.normalize()

df["Время"] = pd.to_timedelta(df["Время"].astype(str), errors="coerce")

df = df[["Имя", "Начало", "Вид временного события", "Время"]].dropna()

# ==== 3. Сортировка ====
df = df.sort_values(["Имя", "Начало", "Время"])

# ==== 4. Предыдущее событие ====
df["prev_type"] = df.groupby(["Имя", "Начало"])["Вид временного события"].shift(1)
df["prev_time"] = df.groupby(["Имя", "Начало"])["Время"].shift(1)

# ==== 5. Маска "Уход → Приход" ====
mask_gap = (
    df["Вид временного события"].str.lower().str.strip().eq("приход") &
    df["prev_type"].str.lower().str.strip().eq("уход") &
    df["prev_time"].notna()
)

# ==== 6. Рассчёт интервалов отсутствия ====
df["Отсутствие"] = pd.Timedelta(0)
df.loc[mask_gap, "Отсутствие"] = df.loc[mask_gap, "Время"] - df.loc[mask_gap, "prev_time"]

df.loc[df["Отсутствие"] < pd.Timedelta(0), "Отсутствие"] = pd.Timedelta(0)

# ==== 7. Агрегация ====
result = (
    df.groupby(["Имя", "Начало"], as_index=False)["Отсутствие"]
      .sum()
      .rename(columns={"Отсутствие": "Время отсутствия"})
)

# Приводим в чч:мм:сс
result["Время отсутствия"] = result["Время отсутствия"].astype("timedelta64[s]")
result["Время отсутствия"] = pd.to_timedelta(result["Время отсутствия"], unit="s")

# ==== 8. Запись в Excel ====
book = load_workbook(excel_file)

with pd.ExcelWriter(excel_file, engine="openpyxl", mode="a", if_sheet_exists="replace") as writer:
    writer.book = book
    result.to_excel(writer, sheet_name=result_sheet, index=False)
