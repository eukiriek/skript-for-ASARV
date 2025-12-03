import pandas as pd

# ---------- 1. Справочник ----------
df_ref = pd.read_excel("shtatka_go.xlsx")

# ---------- 2. Нормализация ключей ----------
df["Сотрудник"] = normalize_tab(df["Сотрудник"].astype(str))
df_ref["ТабНомер"] = normalize_tab(df_ref["ТабНомер"].astype(str))

# ---------- 3. Даты ----------
df["Календарный день"] = (
    pd.to_datetime(df["Календарный день"], errors="coerce", dayfirst=True)
      .dt.normalize()  # оставляем только дату
)

df_ref["ДатаНазнНаДолжность"] = (
    pd.to_datetime(df_ref["ДатаНазнНаДолжность"], errors="coerce", dayfirst=True)
      .dt.normalize()
)
df_ref["ДатаУвольнения"] = (
    pd.to_datetime(df_ref["ДатаУвольнения"], errors="coerce", dayfirst=True)
      .dt.normalize()
)

# границы, чтобы не отсеять всё нафиг
min_date = pd.Timestamp("1900-01-01")
max_date = pd.Timestamp("2099-12-31")

df_ref["ДатаНазнНаДолжность"] = df_ref["ДатаНазнНаДолжность"].fillna(min_date)
df_ref["ДатаУвольнения"] = df_ref["ДатаУвольнения"].fillna(max_date)

# ---------- 4. Подразделения как текст ----------
# если там были числа типа 101, приводим к строке
df_ref["Подразд-е уровень 2"] = (
    clean_str(df_ref["Подразд-е уровень 2"].astype(str))
)
df_ref["Подразд-е уровень 3"] = (
    clean_str(df_ref["Подразд-е уровень 3"].astype(str))
)

# ---------- 5. Оставляем только нужные поля ----------
ref_keep = [
    "ТабНомер",
    "ДатаНазнНаДолжность",
    "ДатаУвольнения",
    "Подразд-е уровень 2",
    "Подразд-е уровень 3",
]
df_ref = df_ref[ref_keep].copy()

# ---------- 6. row_id в df ----------
df = df.reset_index(drop=True)
df["row_id"] = df.index

# ---------- 7. Мердж по табельному ----------
merged = df.merge(
    df_ref,
    how="left",
    left_on="Сотрудник",
    right_on="ТабНомер",
    suffixes=("", "_ref"),
)

print("Совпало по табельному номеру:", merged["ТабНомер"].notna().sum())

# ---------- 8. Фильтр по полному диапазону дат ----------
mask_full = (
    (merged["Календарный день"] >= merged["ДатаНазнНаДолжность"]) &
    (merged["Календарный день"] <= merged["ДатаУвольнения"])
)

print("Совпало по табельному и полному диапазону дат:", mask_full.sum())

candidates = merged.loc[
    mask_full,
    ["row_id", "ДатаНазнНаДолжность", "Подразд-е уровень 2", "Подразд-е уровень 3"]
].copy()

# ЕСЛИ по полному диапазону дат ничего не нашлось — fallback:
# "последнее назначение, которое началось не позже календарного дня"
if candidates.empty:
    print("По полному диапазону дат 0 совпадений — используем fallback по дате назначения.")
    mask_start_only = (
        merged["Календарный день"] >= merged["ДатаНазнНаДолжность"]
    )
    candidates = merged.loc[
        mask_start_only,
        ["row_id", "ДатаНазнНаДолжность", "Подразд-е уровень 2", "Подразд-е уровень 3"]
    ].copy()

# выбираем последнее назначение по дате
candidates = (
    candidates
    .sort_values(["row_id", "ДатаНазнНаДолжность"])
    .drop_duplicates(subset="row_id", keep="last")
    .set_index("row_id")
)

# ---------- 9. Джоиним обратно к df ----------
df = df.drop(columns=["Подразд-е уровень 2", "Подразд-е уровень 3"], errors="ignore")
df = df.join(candidates[["Подразд-е уровень 2", "Подразд-е уровень 3"]], on="row_id")

# финальный тип — строковый (на случай, если pandas всё ещё хочет сделать числа)
df["Подразд-е уровень 2"] = df["Подразд-е уровень 2"].astype("string")
df["Подразд-е уровень 3"] = df["Подразд-е уровень 3"].astype("string")

print("-- добавление СП2/СП3 завершено. Обновлено строк:", candidates.shape[0])
