import pandas as pd

# читаем файл
df = pd.read_excel("your_file.xlsx")

# 1. Конвертируем все нужные поля в timedelta
df["отработанное время_td"] = pd.to_timedelta(df["отработанное время"], errors="coerce")
df["время выхода_td"] = pd.to_timedelta(df["время выхода"], errors="coerce")
df["время входа_td"] = pd.to_timedelta(df["время входа"], errors="coerce")
df["поле 1_td"] = pd.to_timedelta(df["поле 1"], errors="coerce")
df["поле 2_td"] = pd.to_timedelta(df["поле 2"], errors="coerce")

# 2. Максимум из двух полей (перерывы) по строкам
break_max = pd.concat([df["поле 1_td"], df["поле 2_td"]], axis=1).max(axis=1)

# 3. Создаём пустой столбец под новое время
df["новое отработанное время_td"] = pd.NaT

# 4. Условие: если "отработанное время" > 25 часов → оставляем как есть
mask_big = df["отработанное время_td"] > pd.Timedelta(hours=25)
df.loc[mask_big, "новое отработанное время_td"] = df.loc[mask_big, "отработанное время_td"]

# 5. Иначе считаем:
# новое = время выхода - время входа - максимум(поле 1, поле 2)
mask_small = ~mask_big

df.loc[mask_small, "новое отработанное время_td"] = (
    df.loc[mask_small, "время выхода_td"]
    - df.loc[mask_small, "время входа_td"]
    - break_max[mask_small]
)

# 6. Переводим итог в строковый формат чч:мм:сс
df["новое отработанное время"] = (
    df["новое отработанное время_td"]
    .dt.total_seconds()
    .fillna(0)  # если хочешь оставить пусто, убери эту строку
    .astype(int)
)

# Форматируем в HH:MM:SS
df["новое отработанное время"] = df["новое отработанное время"].apply(
    lambda x: f"{x // 3600:02d}:{(x % 3600) // 60:02d}:{x % 60:02d}"
)

# при желании можно удалить технические *_td поля
df = df.drop(columns=["отработанное время_td",
                      "время выхода_td",
                      "время входа_td",
                      "поле 1_td",
                      "поле 2_td",
                      "новое отработанное время_td"])

df.to_excel("your_file_updated.xlsx", index=False)
print("Готово, новое отработанное время посчитано.")
