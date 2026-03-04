import pandas as pd

ROLEVKA_FILE = "rolevka.xlsx"
REF_RF_FILE = "shtatka_rf.xlsx"
REF_GO_FILE = "shtatka_go.xlsx"

OUT_ROLEVKA = "rolevka_new.xlsx"
OUT_UPDATED = "updated.data.xlsx"


# ----------------------------
# 1) Справочники
# ----------------------------
df_ref_rf = pd.read_excel(REF_RF_FILE)
df_ref_go = pd.read_excel(REF_GO_FILE)


# ----------------------------
# 2) Обработка листа rolevka -> мердж + Изменение
# ----------------------------
def process_rolevka_sheet(rolevka_file: str, rolevka_sheet: str, ref_df: pd.DataFrame) -> pd.DataFrame:
    df = pd.read_excel(rolevka_file, sheet_name=rolevka_sheet)

    df_ref_100 = ref_df.loc[ref_df["Рабочеевремя"] == 100, ["ИДДожности", "ФИО", "ТабНомер", "Электроннаяпочта"]].copy()

    # MERGE
    df = df.merge(df_ref_100, on="ИДДожности", how="left", suffixes=("_x", "_y"))

    # Изменение (векторно, без apply)
    df["Изменение"] = df["ФИО_y"].where(df["ФИО_x"] != df["ФИО_y"], other=pd.NA)

    return df


# ----------------------------
# 3) Сохраняем rolevka_new.xlsx одним Writer'ом (без append)
# ----------------------------
df_rf = process_rolevka_sheet(ROLEVKA_FILE, "РФ", df_ref_rf)
df_go = process_rolevka_sheet(ROLEVKA_FILE, "ССП ГО", df_ref_go)
df_full = process_rolevka_sheet(ROLEVKA_FILE, "Полный доступ", df_ref_go)
df_mgmt = process_rolevka_sheet(ROLEVKA_FILE, "Руководство", df_ref_go)

with pd.ExcelWriter(OUT_ROLEVKA, engine="openpyxl") as writer:
    df_rf.to_excel(writer, sheet_name="РФ", index=False)
    df_go.to_excel(writer, sheet_name="ГО", index=False)
    df_full.to_excel(writer, sheet_name="Полный доступ", index=False)
    df_mgmt.to_excel(writer, sheet_name="Руководство", index=False)

print("rolevka_new.xlsx создан (РФ/ГО/Полный доступ/Руководство)")


# ----------------------------
# 4) Build updated sheet (только строки с Изменение)
# ----------------------------
def find_ad_group_column(columns) -> str:
    # Ищем колонку вида "группа в ad" (любые пробелы/регистр)
    for c in columns:
        s = str(c).strip().lower().replace("  ", " ")
        if s == "группа в ad":
            return c

    # запасной вариант: колонка содержит "группа" и "ad"
    for c in columns:
        s = str(c).strip().lower()
        if ("группа" in s) and ("ad" in s):
            return c

    return ""


def build_updates_sheet(source_file: str, source_sheet: str, target_division: str) -> pd.DataFrame:
    df_new = pd.read_excel(source_file, sheet_name=source_sheet)

    # только изменения
    df_changes = df_new[df_new["Изменение"].notna()].copy()

    # найдём "группа в ad"
    ad_col = find_ad_group_column(df_new.columns)
    if not ad_col:
        raise KeyError(f"Не найдена колонка 'группа в ad' на листе '{source_sheet}' в {source_file}")

    # даже если df_changes пустой — вернём DataFrame с нужными колонками (чтобы лист создался)
    updated = pd.DataFrame({
        "подразделение": [target_division] * len(df_changes),
        "должность": df_changes["Должность"].values if len(df_changes) else [],
        "фио": df_changes["ФИО_y"].values if len(df_changes) else [],
        "группа в ад": df_changes[ad_col].values if len(df_changes) else [],
        "табномер": df_changes["ТабНомер"].values if len(df_changes) else [],
    })

    # гарантируем порядок колонок
    updated = updated[["подразделение", "должность", "фио", "группа в ад", "табномер"]]
    return updated


# ----------------------------
# 5) Создаём updated.data.xlsx (ВСЕГДА с INFO листом)
# ----------------------------
sheets_map = [
    ("РФ", "РФ", "ССПГО"),
    ("ГО", "ГО", "ГО"),
    ("Полный доступ", "Полный доступ", "Полный доступ"),
    ("Руководство", "Руководство", "Руководство"),
]

with pd.ExcelWriter(OUT_UPDATED, engine="openpyxl") as writer:
    # ВАЖНО: СРАЗУ пишем INFO, чтобы книга точно имела хотя бы один видимый лист
    info_df = pd.DataFrame({"Сообщение": ["Файл сформирован. Если в листах нет строк — значит изменений не найдено."]})
    info_df.to_excel(writer, sheet_name="INFO", index=False)

    # Пишем все листы (даже если пустые — с заголовками)
    for src_sheet, out_sheet, division in sheets_map:
        df_upd = build_updates_sheet(OUT_ROLEVKA, src_sheet, division)
        df_upd.to_excel(writer, sheet_name=out_sheet, index=False)

print(f"{OUT_UPDATED} создан (INFO + листы с изменениями).")
