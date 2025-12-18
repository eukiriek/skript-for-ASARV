import os
import pandas as pd

def apply_exceptions(
        df,
        sheet_name,
        target_col,
        sprav_file="spravochnik.xlsx",
        exceptions_file="exceptions_new.xlsx"
    ):
    """
    Удаление строк по справочнику исключений.
    Удалённые строки накапливаются в файле exceptions_new.xlsx
    на листе с названием sheet_name.
    """
    # читаем справочник
    df_ref = pd.read_excel(sprav_file, sheet_name=sheet_name)

    # чистим строки
    df[target_col] = clean_str(df[target_col])
    df_ref["exception"] = clean_str(df_ref["exception"])

    # ищем совпадения
    match_values = set(df_ref["exception"].unique())
    mask = df[target_col].isin(match_values)

    # строки-исключения
    df_exceptions_removed = df.loc[mask].copy()

    # если есть что сохранять — пишем/дописываем в exceptions_new.xlsx
    if not df_exceptions_removed.empty:
        if os.path.exists(exceptions_file):
            # файл уже есть → дописываем лист, можно заменить существующий
            with pd.ExcelWriter(
                exceptions_file,
                mode="a",
                engine="openpyxl",
                if_sheet_exists="replace"
            ) as writer:
                df_exceptions_removed.to_excel(
                    writer,
                    sheet_name=sheet_name,
                    index=False
                )
        else:
            # файла ещё нет → создаём новый, без if_sheet_exists
            with pd.ExcelWriter(
                exceptions_file,
                mode="w",
                engine="openpyxl"
            ) as writer:
                df_exceptions_removed.to_excel(
                    writer,
                    sheet_name=sheet_name,
                    index=False
                )

    # возвращаем df без исключений
    return df.loc[~mask].copy()
