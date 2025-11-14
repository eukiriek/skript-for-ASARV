import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем все нужные поля
df["время_выхода"] = pd.to_datetime(df["время_выхода"], errors="coerce")
df["время_входа"] = pd.to_datetime(df["время_входа"], errors="coerce")
df["отработанное"] = pd.to_timedelta(df["отработанное"].astype(str), errors="coerce")
df["поле1"] = pd.to_timedelta(df["поле1"].astype(str), errors="coerce")
df["поле2"] = pd.to_timedelta(df["поле2"].astype(str), errors="coerce")

# Новое поле
df["новое_отработанное"] = pd.NaT

# Условие 1 — если отработанное > 25 часов
mask_big = df["отработанное"] > pd.Timedelta(hours=25)
df.loc[mask_big, "новое_отработанное"] = df.loc[mask_big, "отработанное"]

# Условие 2 — все остальные
mask_small = ~mask_big

df.loc[mask_small, "новое_отработанное"] = (
    df.loc[mask_small, "время_выхода"]
    - df.loc[mask_small, "время_входа"]
    - df.loc[mask_small, ["поле1", "поле2"]].max(axis=1)
)

df.to_excel("result.xlsx", index=False)
print("Готово!")
