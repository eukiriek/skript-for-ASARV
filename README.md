
import pandas as pd

# --- Загружаем основной файл ---
df = pd.read_excel("main.xlsx")

# --- Загружаем файл с праздниками ---
df_holidays = pd.read_excel("prazdnik.xlsx")

# Приводим даты в datetime
df["кд"] = pd.to_datetime(df["кд"], errors="coerce")
df_holidays["праздник"] = pd.to_datetime(df_holidays["праздник"], errors="coerce")

# Приводим поле "норма" к timedelta (время)
df["норма"] = pd.to_timedelta(df["норма"].astype(str), errors="coerce")

# Список всех праздничных дат
holidays = set(df_holidays["праздник"].dropna())

# --- Логика изменения нормы ---
mask = df["кд"].isin(holidays)

# если дата = праздник → норма = норма - 1 час
df.loc[mask, "норма"] = df.loc[mask, "норма"] - pd.Timedelta(hours=1)

# Возвращаем формат ЧЧ:ММ:СС
df["норма"] = df["норма"].dt.strftime("%H:%M:%S")

# --- Сохраняем результат ---
df.to_excel("main_updated.xlsx", index=False)

print("Готово! Поле 'норма' скорректировано для праздничных дат.")
