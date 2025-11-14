import pandas as pd

# Загружаем основную таблицу
df_main = pd.read_excel("main.xlsx")

# Загружаем файл праздников
df_holiday = pd.read_excel("праздник.xlsx")

# Приводим оба столбца к формату даты
df_main["N"] = pd.to_datetime(df_main["N"], errors="coerce")
df_holiday["M"] = pd.to_datetime(df_holiday["M"], errors="coerce")

# Создаем множество праздников (быстрее для поиска)
holiday_set = set(df_holiday["M"].dropna().unique())

# Создаем поле флаг
df_main["флаг"] = df_main["N"].isin(holiday_set).astype(int)

# Сохраняем результат
df_main.to_excel("main_with_flag.xlsx", index=False)

print("Файл main_with_flag.xlsx создан. Поле 'флаг' добавлено.")




import pandas as pd

# -------- 1. Загружаем данные --------
df = pd.read_excel("main.xlsx")                   # основная таблица
df_h = pd.read_excel("праздник.xlsx")             # таблица праздников

# -------- 2. Приводим даты к формату datetime --------
df["N"] = pd.to_datetime(df["N"], errors="coerce", dayfirst=True)
df_h["M"] = pd.to_datetime(df_h["M"], errors="coerce", dayfirst=True)

# -------- 3. Создаём флаг праздника --------
holiday_set = set(df_h["M"].dropna().dt.date)

df["флаг"] = df["N"].dt.date.isin(holiday_set).astype(int)

# -------- 4. Преобразуем норму в timedelta --------
df["норма"] = pd.to_timedelta(df["норма"].astype(str), errors="coerce")

# -------- 5. Если флаг = 1 → уменьшаем норму на 1 час --------
df.loc[df["флаг"] == 1, "норма"] = df.loc[df["флаг"] == 1, "норма"] - pd.Timedelta(hours=1)

# -------- 6. Переводим норму обратно в чч:мм:сс --------
df["норма"] = df["норма"].dt.components.apply(
    lambda r: f"{r.hours:02d}:{r.minutes:02d}:{r.seconds:02d}", axis=1
)

# -------- 7. Сохраняем результат --------
df.to_excel("main_updated.xlsx", index=False)

print("Готово! Поле флаг создано, норма пересчитана.")
