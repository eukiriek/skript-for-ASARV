import pandas as pd

def apply_exceptions(df, sheet_name, target_col, sprav_file="spravochnik.xlsx", ref_col="exception",
                     out_file="exception.xlsx"):
    """
    Удаление строк по справочнику исключений.
    Удалённые строки сохраняются в out_file на отдельный лист с именем sheet_name.
    """

    df_ref = pd.read_excel(sprav_file, sheet_name=sheet_name)

    # если в справочнике нет колонки 'exception', используем ref_col или первую колонку
    if ref_col not in df_ref.columns:
        ref_col = df_ref.columns[0]

    # для ID лучше нормализовать как табельный/числовой ключ (убирает .0 и т.п.)
    df[target_col] = normalize_tab(df[target_col])
    df_ref[ref_col] = normalize_tab(df_ref[ref_col])

    match_values = set(df_ref[ref_col].dropna().unique())
    mask = df[target_col].isin(match_values)

    df_removed = df.loc[mask].copy()

    # пишем в exception.xlsx, не затирая другие листы
    if not df_removed.empty:
        mode = "a" if pd.io.common.file_exists(out_file) else "w"
        with pd.ExcelWriter(out_file, engine="openpyxl", mode=mode, if_sheet_exists="replace") as writer:
            df_removed.to_excel(writer, sheet_name=sheet_name, index=False)

    return df.loc[~mask].copy()
