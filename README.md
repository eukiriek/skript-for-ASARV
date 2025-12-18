import os
import zipfile
from openpyxl.utils.exceptions import InvalidFileException
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
    Удалённые строки накапливаются в exceptions_file на листе sheet_name.
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

    # --- сохраняем исключения в exceptions_new.xlsx ---
    if not df_exceptions_removed.empty:
        if os.path.exists(exceptions_file):
            # файл уже есть: пробуем дописать лист
            try:
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
            except (zipfile.BadZipFile, InvalidFileException):
                # если файл битый / "не zip" — создаём заново
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
        else:
            # файла ещё нет — создаём
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

    # --- возвращаем df без исключений ---
    return df.loc[~mask].copy()
