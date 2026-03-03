# === ДОБАВИТЬ СРАЗУ ПОСЛЕ БЛОКА РФ (после print('работа по РФ завершена')) ===

# 1) Читаем готовый лист РФ из rolevka_new.xlsx (там уже есть Изменение)
df_rf_new = pd.read_excel("rolevka_new.xlsx", sheet_name="РФ")

# 2) Берем только строки, где есть изменения
df_changes = df_rf_new[df_rf_new["Изменение"].notna()].copy()

# 3) Находим колонку "группа в ad" (на случай разного регистра/написания)
ad_group_col = None
for c in df_changes.columns:
    if str(c).strip().lower() == "группа в ad":
        ad_group_col = c
        break

if ad_group_col is None:
    raise KeyError("Не найдена колонка 'группа в ad' на листе РФ в rolevka_new.xlsx")

# 4) Формируем таблицу для updated.data.xlsx
updated_rf = pd.DataFrame({
    "подразделение": "ССПГО",
    "должность": df_changes["Должность"],
    "фио": df_changes["ФИО_y"],
    "группа в ад": df_changes[ad_group_col],
    "табномер": df_changes["ТабНомер"],
})

# 5) Сохраняем новый файл updated.data.xlsx с листом РФ
updated_rf.to_excel("updated.data.xlsx", sheet_name="РФ", index=False)

print("updated.data.xlsx создан (лист РФ заполнен изменениями).")
