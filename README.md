import pandas as pd
import numpy as np

# --- читаем справочники ---
df_otpusk = pd.read_excel("otpusk.xlsx")          # Табельный номер | Всего
df_sprav = pd.read_excel("spravoshnik.xlsx")      # Табельный номер | ДопОтпуск | Северные

# --- приводим ключи ---
df["Таб Номер"] = df["Таб Номер"].astype(str)
df_otpusk["Табельный номер"] = df_otpusk["Табельный номер"].astype(str)
df_sprav["Табельный номер"] = df_sprav["Табельный номер"].astype(str)

# --- маппинги ---
otpusk_map = df_otpusk.set_index("Табельный номер")["Всего"]

sprav_map = (
    df_sprav
    .set_index("Табельный номер")[["ДопОтпуск", "Северные"]]
    .sum(axis=1)
)

# --- инициализация поля ---
df["Накопленные дни отпуска"] = 0

# --- условия ---
mask_180_730 = (df["Стаж, дней"] > 180) & (df["Стаж, дней"] < 731)
mask_731_plus = df["Стаж, дней"] >= 731

# --- 180–730 дней → otpusk.xlsx ---
df.loc[mask_180_730, "Накопленные дни отпуска"] = (
    df.loc[mask_180_730, "Таб Номер"].map(otpusk_map)
)

# --- 731+ дней → (24 + доп + северные) * 2 ---
df.loc[mask_731_plus, "Накопленные дни отпуска"] = (
    (24 + df.loc[mask_731_plus, "Таб Номер"].map(sprav_map)) * 2
)

# --- если не нашли значение → 0 ---
df["Накопленные дни отпуска"] = df["Накопленные дни отпуска"].fillna(0)

# --- выгрузка ---
df.to_excel("burnout_rate.xlsx", index=False)
