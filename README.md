import pandas as pd

# ==== Настройки (при необходимости поменяйте имена файлов/столбцов) ====
MAIN_FILE = "main.xlsx"
REF_FILE  = "reference.xlsx"

COL_MAIN_DATE = "Дата"        # столбец с датой в основном файле
COL_MAIN_TN   = "ТабНомер"    # столбец с табельным номером в основном файле

COL_REF_LOWER = "Нижняя_граница"  # нижняя граница диапазона в справочнике
COL_REF_UPPER = "Верхняя_граница" # верхняя граница диапазона в справочнике
COL_REF_N     = "N"               # значение N из справочника
COL_REF_M     = "M"               # значение M из справочника

OUT_FILE_MATCHED = "result_match.xlsx"  # куда сохранить результат
# ======================================================================

# 1) Загрузка
df_main = pd.read_excel(MAIN_FILE)
df_ref  = pd.read_excel(REF_FILE)

# 2) Приведение дат
df_main[COL_MAIN_DATE] = pd.to_datetime(df_main[COL_MAIN_DATE], errors="coerce", dayfirst=True)
df_ref[COL_REF_LOWER]  = pd.to_datetime(df_ref[COL_REF_LOWER],  errors="coerce", dayfirst=True)
df_ref[COL_REF_UPPER]  = pd.to_datetime(df_ref[COL_REF_UPPER],  errors="coerce", dayfirst=True)

# Защита от некорректных дат
df_main = df_main.dropna(subset=[COL_MAIN_DATE]).copy()
df_ref  = df_ref.dropna(subset=[COL_REF_LOWER, COL_REF_UPPER]).copy()

# 3) Сортируем для asof-джойна по нижней границе
df_main_sorted = df_main.sort_values(COL_MAIN_DATE).reset_index(drop=True)
df_ref_sorted  = df_ref.sort_values(COL_REF_LOWER).reset_index(drop=True)

# 4) Джойн по принципу:
#    найти последнюю нижнюю границу, не превышающую дату из main,
#    затем отфильтровать те, где дата <= верхней границы (попадание в интервал).
merged = pd.merge_asof(
    df_main_sorted,
    df_ref_sorted[[COL_REF_LOWER, COL_REF_UPPER, COL_REF_N, COL_REF_M]],
    left_on=COL_MAIN_DATE,
    right_on=COL_REF_LOWER,
    direction="backward",
    allow_exact_matches=True
)

# 5) Фильтр по условию попадания в интервал [LOWER; UPPER] включительно
in_range_mask = merged[COL_MAIN_DATE].le(merged[COL_REF_UPPER])

# Если есть пересечения интервалов с одинаковыми LOWER, asof возьмёт ближайший LOWER.
# При необходимости можно дополнительно проверить, что LOWER <= date (asof это уже гарантирует).
matched = merged[in_range_mask].copy()

# 6) Собираем итоговые поля:
#    ТабНомер (из main), N и M (из reference). 
#    При желании добавьте ещё поля из main/reference.
result = matched[[COL_MAIN_TN, COL_MAIN_DATE, COL_REF_N, COL_REF_M]].rename(
    columns={
        COL_MAIN_TN: "ТабНомер",
        COL_MAIN_DATE: "Дата",
        COL_REF_N: "N_ref",
        COL_REF_M: "M_ref",
    }
)

# 7) Сохраняем результат
result.to_excel(OUT_FILE_MATCHED, index=False)

print(f"Найдено совпадений по диапазонам: {len(result)}")
print(f"Результат сохранён в: {OUT_FILE_MATCHED}")
