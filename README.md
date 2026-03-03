import pandas as pd

# --- Читаем справочники ---
df_ref_rf = pd.read_excel('shtatka_rf.xlsx')
df_ref_go = pd.read_excel('shtatka_go.xlsx')

# --- Функция обработки листа rolevka: мердж + поле Изменение ---
def process_rolevka_sheet(rolevka_file: str, sheet_name: str, ref_df: pd.DataFrame) -> pd.DataFrame:
    df = pd.read_excel(rolevka_file, sheet_name=sheet_name)

    df_ref_100 = ref_df[ref_df['Рабочеевремя'] == 100].copy()
    df_ref_100 = df_ref_100[['ИДДожности', 'ФИО', 'ТабНомер', 'Электроннаяпочта']]

    df = df.merge(df_ref_100, on='ИДДожности', how='left')

    # Создаем поле Изменение
    df['Изменение'] = df.apply(
        lambda row: row['ФИО_y'] if row['ФИО_x'] != row['ФИО_y'] else None,
        axis=1
    )
    return df


# --- 1) Формируем rolevka_new.xlsx со всеми листами ---
rolevka_file = "rolevka.xlsx"
out_rolevka = "rolevka_new.xlsx"

# РФ
df_rf = process_rolevka_sheet(rolevka_file, "РФ", df_ref_rf)
df_rf.to_excel(out_rolevka, sheet_name='РФ', index=False)
print('работа по РФ завершена')

# ГО
df_go = process_rolevka_sheet(rolevka_file, "ССП ГО", df_ref_go)
with pd.ExcelWriter(out_rolevka, mode='a', engine='openpyxl', if_sheet_exists='replace') as writer:
    df_go.to_excel(writer, sheet_name='ГО', index=False)
print('работа по категории ГО завершена')

# Полный доступ
df_full = process_rolevka_sheet(rolevka_file, "Полный доступ", df_ref_go)
with pd.ExcelWriter(out_rolevka, mode='a', engine='openpyxl', if_sheet_exists='replace') as writer:
    df_full.to_excel(writer, sheet_name='Полный доступ', index=False)
print('работа по Полный доступ завершена')

# Руководство
df_mgmt = process_rolevka_sheet(rolevka_file, "Руководство", df_ref_go)
with pd.ExcelWriter(out_rolevka, mode='a', engine='openpyxl', if_sheet_exists='replace') as writer:
    df_mgmt.to_excel(writer, sheet_name='Руководство', index=False)
print('работа по категории Руководство завершена')


# --- 2) Формируем updated.data.xlsx (только строки с Изменение) ---
def build_updates_sheet(source_file: str, source_sheet: str, target_division: str) -> pd.DataFrame:
    df_new = pd.read_excel(source_file, sheet_name=source_sheet)

    df_changes = df_new[df_new["Изменение"].notna()].copy()
    if df_changes.empty:
        return pd.DataFrame(columns=["подразделение", "должность", "фио", "группа в ад", "табномер"])

    # Ищем колонку "группа в ad" без привязки к регистру/пробелам
    ad_group_col = None
    for c in df_changes.columns:
        if str(c).strip().lower() == "группа в ad":
            ad_group_col = c
            break
    if ad_group_col is None:
        raise KeyError(f"Не найдена колонка 'группа в ad' на листе '{source_sheet}' в {source_file}")

    updated = pd.DataFrame({
        "подразделение": target_division,
        "должность": df_changes["Должность"],
        "фио": df_changes["ФИО_y"],
        "группа в ад": df_changes[ad_group_col],
        "табномер": df_changes["ТабНомер"],
    })
    return updated


source = out_rolevka
out_file = "updated.data.xlsx"

sheets_map = [
    ("РФ", "РФ", "ССПГО"),  # лист-источник, лист-выход, значение "подразделение"
    ("ГО", "ГО", "ГО"),
    ("Полный доступ", "Полный доступ", "Полный доступ"),
    ("Руководство", "Руководство", "Руководство"),
]

# Пишем все листы в один файл
with pd.ExcelWriter(out_file, engine="openpyxl") as writer:
    for src_sheet, out_sheet, division in sheets_map:
        df_upd = build_updates_sheet(source, src_sheet, division)
        df_upd.to_excel(writer, sheet_name=out_sheet, index=False)

print(f"Файл {out_file} создан: изменения разнесены по листам.")
