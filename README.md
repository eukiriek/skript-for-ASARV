import pandas as pd

# основная таблица
df_main = pd.read_excel("main.xlsx")

# добавочная таблица
df_add = pd.read_excel("add.xlsx")

# на всякий случай приводим ключи к одному типу
df_main["Табномер"] = df_main["Табномер"].astype(str).str.strip()
df_add["Табномер"]  = df_add["Табномер"].astype(str).str.strip()

df_main["День"] = pd.to_datetime(df_main["День"], dayfirst=True, errors="coerce")
df_add["День"]  = pd.to_datetime(df_add["День"],  dayfirst=True, errors="coerce")

# подтягиваем поле "Новые переработки" из добавочной таблицы по двум ключам
df_main = df_main.merge(
    df_add[["Табномер", "День", "Новые переработки"]],
    on=["Табномер", "День"],
    how="left",
    suffixes=("", "_из_доп")
)

# если в основной уже было поле "Новые переработки" и надо его обновить:
# df_main["Новые переработки"] = df_main["Новые переработки_из_доп"].fillna(df_main["Новые переработки"])
# df_main.drop(columns=["Новые переработки_из_доп"], inplace=True)
