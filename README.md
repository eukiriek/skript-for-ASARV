written_any_sheet = False

with pd.ExcelWriter(out_file, engine="openpyxl") as writer:
    for src_sheet, out_sheet, division in sheets_map:
        df_upd = build_updates_sheet(source, src_sheet, division)

        # Пишем только если есть строки
        if not df_upd.empty:
            df_upd.to_excel(writer, sheet_name=out_sheet, index=False)
            written_any_sheet = True

    # Если вообще нет изменений — создаём технический лист
    if not written_any_sheet:
        pd.DataFrame({"Сообщение": ["Изменений не обнаружено"]}) \
            .to_excel(writer, sheet_name="INFO", index=False)

print(f"Файл {out_file} создан.")
