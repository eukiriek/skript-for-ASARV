import pandas as pd
import numpy as np
from openpyxl import load_workbook

# ========= имя файла в текущей папке =========
excel_file = "data.xlsx"    # ← укажи имя файла, который лежит в той же папке

# ========= лист с исходными данными =========
source_sheet = "Офисы"      # ← поправь на свой лист
result_sheet = "Время_отсутствия"

# ========= 1. читаем Excel =========
df = pd.read_excel(excel_file, sheet_name=source_sheet)

# Приведение типов
df["Имя"] = df["Имя"].astype(str).str.strip()

# Дата дня
df["Начало"] = pd.to_datetime(df["Начало"], errors="coerce", dayfirst=True)
df["Начало"] = df["Начало"].dt.normalize()

# Время события
df["Время"] = pd.to_timedelta(df["Время"].astype(str), errors="coerce")

# Только нужные поля
df = df[["Имя", "Начало", "Вид временного события", "Время"]].dropna()

# ========= 2. сортировка =========
df = df.sort_values(["Имя", "Начало", "Время"])

# ========= 3. предыдущее событие =========
df["prev_type"] = df.groupby(["Имя", "Начало"])["Вид временного события"].shift(1)
df["prev_time"] = df.groupby(["Имя", "Начало"])["Время"].shift(1)

# Маска: "Уход → Приход"
mask_gap = (
    df["Вид временного события"].str.lower().str.strip().eq("приход") &
    df["prev_type"].str.lower().str.strip().eq("уход") &
    df["prev_time"].notna()
)

# ========= 4. считаем интервалы отсутствия =========
df["Отсутствие"] = pd.Timedelta(0)
df.loc[mask_gap, "Отсутствие"] = df.loc[mask_gap, "Время"] - df.loc[mask_gap, "prev_time"]

# Убираем отрицательные интервалы
df.loc[df["Отсутствие"] < pd.Timedelta(0), "Отсутствие"] = pd.Timedelta(0)

# ========= 5. агрегация =========
result = (
    df.groupby(["Имя", "Начало"], as_index=False)["Отсутствие"]
      .sum()
      .rename(columns={"Отсутствие": "Время отсутствия"})
)

# Приводим к формату чч:мм:сс
result["Время отсутствия"] = result["Время отсутствия"].astype("timedelta64[s]")
result["Время отсутствия"] = pd.to_timedelta(result["Время отсутствия"], unit="s")

# ========= 6. запись в файл =========
book = load_workbook(excel_file)

with pd.ExcelWriter(excel_file, engine="openpyxl", mode="a", if_sheet_exists="replace") as writer:
    writer.book = book
    result.to_excel(writer, sheet_name=result_sheet, index=False)
