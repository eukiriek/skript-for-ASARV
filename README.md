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

    df_ref_100 = ref_df.loc[
        ref_df["Рабочеевремя"] == 100,
        ["ИДДожности", "ФИО", "ТабНомер", "Электроннаяпочта"]
    ].copy()

    df = df.merge(df_ref_100, on="ИДДожности", how="left", suffixes=("_x", "_y"))

    # Изменение (векторно)
    df["Изменение"] = df["ФИО_y"].where(df["ФИО_x"] != df["ФИО_y"], other=pd.NA)

    return df


# ----------------------------
# 3) Сохраняем rolevka_new.xlsx одним Writer'ом
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
# 4) Хелперы для updated.data.xlsx
# ----------------------------
def find_ad_group_column(columns) -> str:
    for c in columns:
        s = str(c).strip().lower().replace("  ", " ")
        if s == "группа в ad":
            return c
    for c in columns:
        s = str(c).strip().lower()
        if ("группа" in s) and ("ad" in s):
            return c
    return ""


def resolve_division_series(df_changes: pd.DataFrame, source_sheet: str) -> pd.Series:
    """
    Подразделение берём из rolevka_new.xlsx:
    - для листа РФ: из колонки 'РФ'
    - для листа ГО (источник 'ГО'): из колонки 'ССП ГО'
    - для листа Руководство: из колонки 'Подразделение'
    - для Полный доступ: если нужна логика — поставили по умолчанию 'Полный доступ' (можно поменять)
    """
    if source_sheet == "РФ":
        col = "РФ"
    elif source_sheet == "ГО":
        col = "ССП ГО"
    elif source_sheet == "Руководство":
        col = "Подразделение"
    elif source_sheet == "Полный доступ":
        # если в этом листе есть своя колонка подразделения — скажите название, я подставлю
        return pd.Series(["Полный доступ"] * len(df_changes), index=df_changes.index)
    else:
        return pd.Series([""] * len(df_changes), index=df_changes.index)

    if col not in df_changes.columns:
        raise KeyError(f"Не найдена колонка '{col}' на листе '{source_sheet}' (в rolevka_new.xlsx).")

    return df_changes[col]


def build_updates_sheet(source_file: str, source_sheet: str) -> pd.DataFrame:
    df_new = pd.read_excel(source_file, sheet_name=source_sheet)

    # только изменения
    df_changes = df_new[df_new["Изменение"].notna()].copy()

    # найдём "группа в ad"
    ad_col = find_ad_group_column(df_new.columns)
    if not ad_col:
        raise KeyError(f"Не найдена колонка 'группа в ad' на листе '{source_sheet}' в {source_file}")

    # подразделение — из нужной колонки (зависит от листа)
    division_series = resolve_division_series(df_changes, source_sheet) if not df_changes.empty else pd.Series([], dtype="object")

    # даже если df_changes пустой — вернём DataFrame с нужными колонками (чтобы лист создался)
    updated = pd.DataFrame({
        "подразделение": division_series.values if len(division_series) else [],
        "должность": df_changes["Должность"].values if len(df_changes) else [],
        "фио": df_changes["ФИО_y"].values if len(df_changes) else [],
        "группа в ад": df_changes[ad_col].values if len(df_changes) else [],
        "табномер": df_changes["ТабНомер"].values if len(df_changes) else [],
    })

    updated = updated[["подразделение", "должность", "фио", "группа в ад", "табномер"]]
    return updated


# ----------------------------
# 5) Создаём updated.data.xlsx (всегда с INFO листом)
# ----------------------------
with pd.ExcelWriter(OUT_UPDATED, engine="openpyxl") as writer:
    pd.DataFrame({"Сообщение": ["Файл сформирован. Если в листах нет строк — значит изменений не найдено."]}) \
        .to_excel(writer, sheet_name="INFO", index=False)

    # ВАЖНО: source_sheet здесь — имена листов В rolevka_new.xlsx
    # РФ, ГО, Полный доступ, Руководство
    for sheet in ["РФ", "ГО", "Полный доступ", "Руководство"]:
        df_upd = build_updates_sheet(OUT_ROLEVKA, sheet)
        df_upd.to_excel(writer, sheet_name=sheet, index=False)

print(f"{OUT_UPDATED} создан (INFO + листы с изменениями и корректным 'подразделение').")
