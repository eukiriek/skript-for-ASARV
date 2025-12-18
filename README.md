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
    Удаляет строки из df по справочнику исключений (sprav_file, лист sheet_name).
    Все удалённые строки накапливаются в одном листе "exceptions" файла exceptions_file.
    """

    # --- читаем справочник ---
    df_ref = pd.read_excel(sprav_file, sheet_name=sheet_name)

    # --- чистим строки ---
    df[target_col] = clean_str(df[target_col])
    df_ref["exception"] = clean_str(df_ref["exception"])

    # --- находим совпадения ---
    match_values = set(df_ref["exception"].dropna().unique())
    mask = df[target_col].isin(match_values)

    # строки-исключения, которые надо вынести в отдельный файл
    df_exceptions_removed = df.loc[mask].copy()

    # --- сохраняем все исключения в одном листе ---
    if not df_exceptions_removed.empty:
        # добавим столбец с типом/источником исключения (по желанию можно убрать)
        df_exceptions_removed["Источник_исключения"] = sheet_name

        if os.path.exists(exceptions_file):
            try:
                # читаем уже накопленные исключения
                df_old = pd.read_excel(exceptions_file, sheet_name="exceptions")
                df_all = pd.concat([df_old, df_exceptions_removed], ignore_index=True)
            except Exception:
                # если файл битый или листа нет — начинаем с нуля
                df_all = df_exceptions_removed.copy()
        else:
            df_all = df_exceptions_removed.copy()

        # перезаписываем файл целиком одним листом
        with pd.ExcelWriter(exceptions_file, mode="w", engine="openpyxl") as writer:
            df_all.to_excel(writer, sheet_name="exceptions", index=False)

    # --- возвращаем df без исключений ---
    return df.loc[~mask].copy()
