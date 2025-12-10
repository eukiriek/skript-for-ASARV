import pandas as pd

# ----- 1. НОРМАЛИЗУЕМ КЛЮЧИ -----

# копии на всякий случай
df_main = df_main.copy()
df_add = df_add.copy()

# Табельные номера → строка без пробелов
df_main["Табномер1_norm"] = df_main["Табномер1"].astype(str).str.strip()
df_add["Табномер2_norm"]  = df_add["Табномер2"].astype(str).str.strip()

# Дни → datetime + обрезать время, если оно есть
df_main["День1_norm"] = pd.to_datetime(df_main["День1"], dayfirst=True, errors="coerce").dt.date
df_add["День2_norm"]  = pd.to_datetime(df_add["День2"],  dayfirst=True, errors="coerce").dt.date

# ----- 2. ГОТОВИМ ДОП.ТАБЛИЦУ ДЛЯ MERGE -----

# переименуем нормализованные ключи под ключи основной
df_add_for_merge = df_add.rename(columns={
    "Табномер2_norm": "Табномер1_norm",
    "День2_norm": "День1_norm"
})

# оставляем только ключи + нужное поле
df_add_for_merge = df_add_for_merge[["Табномер1_norm", "День1_norm", "Новые переработки"]]

# ----- 3. MERGE ПО ДВУМ КЛЮЧАМ -----

df_main = df_main.merge(
    df_add_for_merge,
    how="left",
    on=["Табномер1_norm", "День1_norm"]  # здесь совпадения ищутся
)

# теперь в df_main есть столбец "Новые переработки" из доп.таблицы

# ----- 4. (ОПЦИОНАЛЬНО) ЕСЛИ У ТЕБЯ УЖЕ БЫЛО ПОЛЕ "Новые переработки" В ОСНОВНОЙ -----
# и ты хочешь обновлять только там, где нашлись значения:

# df_main["Новые переработки"] = df_main["Новые переработки"].fillna(df_main["Новые переработки_старая"])

# ----- 5. МОЖНО УБРАТЬ ТЕХНИЧЕСКИЕ ПОЛЯ -----

df_main = df_ma
