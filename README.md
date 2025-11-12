import pandas as pd
from pathlib import Path

# ====== Файлы (при необходимости поменяйте имена) ======
MAIN_XLSX = "main.xlsx"         # содержит: Сотрудник, Календарь.дата
REF_XLSX  = "reference.xlsx"    # содержит: Таб номер, Дата приёма, Дата увольнения, Сп1, Сп2
OUT_XLSX  = "result.xlsx"

# ====== Загрузка ======
df_main = pd.read_excel(MAIN_XLSX)
df_ref  = pd.read_excel(REF_XLSX)

# ====== Нормализация названий столбцов (убираем \r/\n и пробелы по краям) ======
df_main.columns = df_main.columns.astype(str).str.replace(r'[\r\n]+', '', regex=True).str.strip()
df_ref.columns  = df_ref.columns.astype(str).str.replace(r'[\r\n]+', '', regex=True).str.strip()

# Проверим, что нужные колонки есть
req_main = {"Сотрудник", "Календарь.дата"}
req_ref  = {"Таб номер", "Дата приёма", "Дата увольнения", "Сп1", "Сп2"}
missing_main = req_main - set(df_main.columns)
missing_ref  = req_ref  - set(df_ref.columns)
if missing_main or missing_ref:
    raise ValueError(f"Нет нужных колонок. В main отсутствуют: {missing_main}; в reference отсутствуют: {missing_ref}")

# ====== Приведение типов ======
# Ключевые поля для сопоставления
df_main["Сотрудник"] = df_main["Сотрудник"].astype(str).str.strip()
df_ref["Таб номер"]  = df_ref["Таб номер"].astype(str).str.strip()

# Даты: в исходных файлах может быть формат dd.mm.yyyy — укажем dayfirst=True
df_main["Календарь.дата"] = pd.to_datetime(df_main["Календарь.дата"], errors="coerce", dayfirst=True)
df_ref["Дата приёма"]     = pd.to_datetime(df_ref["Дата приёма"], errors="coerce", dayfirst=True)
df_ref["Дата увольнения"] = pd.to_datetime(df_ref["Дата увольнения"], errors="coerce", dayfirst=True)

# Если "Дата увольнения" пустая (NaT) — считаем, что сотрудник всё ещё работает, ставим далёкую дату
open_end_date = pd.Timestamp("2099-12-31")
df_ref["Дата увольнения"] = df_ref["Дата увольнения"].fillna(open_end_date)

# На всякий случай уберём строки c пустым табномером в справочнике
df_ref = df_ref[df_ref["Таб номер"].notna() & (df_ref["Таб номер"] != "")].copy()

# ====== Подготовка для однозначного сопоставления ======
# Сохраняем индекс основной таблицы, чтобы после merge восстановить единственность строк
df_main = df_main.reset_index().rename(columns={"index": "row_id"})

# Берём только нужные колонки из справочника
ref_keep = ["Таб номер", "Дата приёма", "Дата увольнения", "Сп1", "Сп2"]
df_ref = df_ref[ref_keep].copy()

# ====== Соединяем по ключу "Сотрудник == Таб номер" ======
merged = df_main.merge(
    df_ref,
    how="left",
    left_on="Сотрудник",
    right_on="Таб номер",
    suffixes=("", "_ref")
)

# ====== Фильтруем по попаданию даты в диапазон ======
in_range = (
    (merged["Календарь.дата"] >= merged["Дата приёма"]) &
    (merged["Календарь.дата"] <= merged["Дата увольнения"])
)

# Оставим только корректные попадания и выберем по одному на каждую строку main
candidates = merged[in_range].copy()

# Если для одной строки main найдено несколько периодов (на случай пересечений),
# можно выбрать самый "близкий" по дате приёма (например, максимальный Дата приёма)
candidates = candidates.sort_values(["row_id", "Дата приёма"])
best_match = candidates.drop_duplicates(subset="row_id", keep="last")

# ====== Собираем финальный датафрейм ======
result = df_main.set_index("row_id").copy()

# По умолчанию создадим пустые столбцы "Сп1" и "Сп2"
result["Сп1"] = pd.NA
result["Сп2"] = pd.NA

# Заполним там, где нашлись подходящие периоды
result.loc[best_match["row_id"].values, ["Сп1", "Сп2"]] = best_match.set_index("row_id")[["Сп1", "Сп2"]].values

# Вернём обычный индекс
result = result.reset_index(drop=True)

# ====== Сохраняем ======
result.to_excel(OUT_XLSX, index=False)

print(f"Готово. Итог сохранён в: {Path(OUT_XLSX).resolve()}")
