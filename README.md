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
    # Читаем справочник для нужного листа
    df_ref = pd.read_excel(sprav_file, sheet_name=sheet_name)

    # Чистим строки
    df[target_col] = clean_str(df[target_col])
    df_ref["exception"] = clean_str(df_ref["exception"])

    # Ищем совпадения
    match_values = set(df_ref["exception"].unique())
    mask = df[target_col].isin(match_values)

    # Строки-исключения (их нужно сохранить отдельно)
    df_exceptions_removed = df.loc[mask].copy()

    # Если есть что сохранять — пишем/дописываем в exceptions_new.xlsx
    if not df_exceptions_removed.empty:
        # если файл уже существует — дописываем лист, иначе создаём новый файл
        mode = "a" if os.path.exists(exceptions_file) else "w"
        with pd.ExcelWriter(exceptions_file, mode=mode,
                            if_sheet_exists="replace") as writer:
            df_exceptions_removed.to_excel(
                writer,
                sheet_name=sheet_name,
                index=False
            )

    # Возвращаем df без исключений
    return df.loc[~mask].copy()
