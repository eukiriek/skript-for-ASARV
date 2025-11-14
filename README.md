import pandas as pd

df = pd.read_excel("your_file.xlsx")

# --- Приводим поля к формату timedelta ---
df["время"] = pd.to_timedelta(df["время"], errors="coerce")
df["N"] = pd.to_numeric(df["N"], errors="coerce")
df["M"] = pd.to_numeric(df["M"], errors="coerce")

# --- Условие: если N=1 и M=0 ---
mask = (df["N"] == 1) & (df["M"] == 0)

df.loc[mask, "время"] = df.loc[mask, "время"] - pd.Timedelta(hours=1)

# --- Возвращаем формат чч:мм:сс ---
df["время"] = df["время"].dt.strftime("%H:%M:%S")

df.to_excel("updated.xlsx", index=False)

print("ГОТОВО.")
