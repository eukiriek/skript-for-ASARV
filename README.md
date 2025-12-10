import pandas as pd

# 1. Приводим ключи к одному виду

# Табельные номера / сотрудники -> строка, убираем пробелы
df["Сотрудник"] = df["Сотрудник"].astype(str).str.strip()
df_add["Табномер"] = df_add["Табномер"].astype(str).str.strip()

# Даты -> datetime с одинаковыми настройками
df["Календарный день"] = pd.to_datetime(
    df["Календарный день"], dayfirst=True, errors="coerce"
)
df_add["Начало"] = pd.to_datetime(
    df_add["Начало"], dayfirst=True, errors="coerce"
)

# 2. Переносим поле "Новое отсутствие" по двум ключам

df = df.merge(
    df_add[["Табномер", "Начало", "Новое отсутствие"]],
    how="left",
    left_on=["Сотрудник", "Календарный день"],  # ключи из основной
    right_on=["Табномер", "Начало"]             # ключи из доп.таблицы
)

# 3. Переименовываем поле и убираем лишние ключи из доп.таблицы

df.rename(columns={"Новое отсутствие": "Новое отсутствие из enter"}, inplace=True)
df.drop(columns=["Табномер", "Начало"], inplace=True)
