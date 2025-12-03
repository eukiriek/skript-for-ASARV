import pandas as pd

# Загружаем данные
df_main = pd.read_excel("main.xlsx")      # основная таблица
df_ref  = pd.read_excel("ref.xlsx")       # доп. таблица

# Убедимся, что ключи — целые числа
df_main["Сотрудник"] = pd.to_numeric(df_main["Сотрудник"], errors="coerce").astype("Int64")
df_ref["id"]         = pd.to_numeric(df_ref["id"], errors="coerce").astype("Int64")

# Мерж по ключам: Сотрудник == id
df_merged = df_main.merge(
    df_ref[["id", "сп"]],
    left_on="Сотрудник",
    right_on="id",
    how="left"
)

# можно удалить вспомогательное поле id
df_merged = df_merged.drop(columns=["id"])

# Итог: df_merged содержит поле "сп" из доп.таблицы
