import pandas as pd
import numpy as np
from openpyxl import load_workbook

# ========= настройки =========
file_path = "data.xlsx"          # путь к файлу
source_sheet = "Офисы"           # лист с исходными данными (поправь под себя)
result_sheet = "Время_отсутствия"  # имя листа для результата

# ========= 1. читаем данные =========
df = pd.read_excel(file_path, sheet_name=source_sheet)

# приведение типов
df["Имя"] = df["Имя"].astype(str).str.strip()

# дата дня (поле "Начало")
df["Начало"] = pd.to_datetime(df["Начало"], errors="coerce", dayfirst=True)
df["Начало"] = df["Начало"].dt.normalize()   # только дата, время = 00:00

# время события
df["Время"] = pd.to_timedelta(df["Время"].astype(str), errors="coerce")

# оставим только нужные поля и уберём пустые
df = df[["Имя", "Начало", "Вид временного события", "Время"]].dropna()

# ========= 2. сортировка =========
df = df.sort_values(["Имя", "Начало", "Время"])

# ========= 3. предыдущие события внутри дня =========
df["prev_type"] = df.groupby(["Имя", "Начало"])["Вид временного события"].shift(1)
df["prev_time"] = df.groupby(["Имя", "Начало"])["Время"].shift(1)

# маска: сейчас Приход, до этого был Уход
mask_gap = (
    df["Вид временного события"].str.lower().str.strip().eq("приход") &
    df["prev_type"].str.lower().str.strip().eq("уход") &
    df["prev_time"].notna()
)

# ========= 4. считаем интервалы отсутствия =========
df["Отсутствие"] = pd.Timedelta(0)
df.loc[mask_gap, "Отсутствие"] = df.loc[mask_gap, "Время"] - df.loc[mask_gap, "prev_time"]

# на всякий случай убираем отрицательные интервалы
df.loc[df["Отсутствие"] < pd.Timedelta(0), "Отсутствие"] = pd.Timedelta(0)

# ========= 5. агрегация по человеку и дню =========
result = (
    df.groupby(["Имя", "Начало"], as_index=False)["Отсутствие"]
      .sum()
      .rename(columns={"Отсутствие": "Время отсутствия"})
)

# (по желанию) строковый формат чч:мм:сс
result["Время отсутствия"] = result["Время отсутствия"].astype("timedelta64[s]")
result["Время отсутствия"] = pd.to_timedelta(result["Время отсутствия"], unit="s")

# ========= 6. запись в новый лист того же файла =========
book = load_workbook(file_path)
with pd.ExcelWriter(file_path, engine="openpyxl", mode="a", if_sheet_exists="replace") as writer:
    writer.book = book
    result.to_excel(writer, sheet_name=result_sheet, index=False)
