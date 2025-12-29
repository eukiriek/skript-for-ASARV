import pandas as pd

# --- функция нормализации табельного номера ---
def normalize_tab(series):
    """
    Приводит табельные номера к единому виду:
    10, 10.0, '10 ', '10.0' -> '10'
    всё, что не число, превращается в пустую строку.
    """
    num = pd.to_numeric(series, errors="coerce")
    return num.apply(lambda x: "" if pd.isna(x) else str(int(x)))

# --- чтение файлов ---
df = pd.read_excel("burnout_source.xlsx")  # основной файл, где считаем стаж
df_otpusk = pd.read_excel("otpusk.xlsx")   # Табельный номер | Всего
df_sprav = pd.read_excel("spravochnik.xlsx", sheet_name="add_vac")  # Табельный номер | ДопДни | Северные

# --- нормализуем табельные номера во всех таблицах ---
df["Таб Номер"] = normalize_tab(df["Таб Номер"])
df_otpusk["Табельный номер"] = normalize_tab(df_otpusk["Табельный номер"])
df_sprav["Табельный номер"] = normalize_tab(df_sprav["Табельный номер"])

# --- готовим мапы с УНИКАЛЬНЫМ индексом ---
# otpusk: суммируем "Всего" по каждому табельному
otpusk_map = (
    df_otpusk
    .groupby("Табельный номер", as_index=True)["Всего"]
    .sum()
)

# spravochnik: (ДопДни + Северные) по каждому табельному
df_sprav[["ДопДни", "Северные"]] = df_sprav[["ДопДни", "Северные"]].fillna(0)
df_sprav["доп_все"] = df_sprav["ДопДни"] + df_sprav["Северные"]

sprav_map = (
    df_sprav
    .groupby("Табельный номер", as_index=True)["доп_все"]
    .sum()
)

# --- расчёт накопленных дней отпуска в основном df ---
df["Накопленные дни отпуска"] = 0

mask_180_730 = (df["Стаж, дни"] > 180) & (df["Стаж, дни"] < 731)
mask_731_plus = df["Стаж, дни"] >= 731

# 1) стаж 180–730 дней: берём из otpusk.xlsx
df.loc[mask_180_730, "Накопленные дни отпуска"] = (
    df.loc[mask_180_730, "Таб Номер"].map(otpusk_map)
)

# 2) стаж 731+ дней: (24 + доп_все) * 2
df.loc[mask_731_plus, "Накопленные дни отпуска"] = (
    24 + df.loc[mask_731_plus, "Таб Номер"].map(sprav_map)
) * 2

# если где-то не нашлось значения — ставим 0
df["Накопленные дни отпуска"] = df["Накопленные дни отпуска"].fillna(0)

# --- сохранение результата ---
df.to_excel("burnout_rate.xlsx", index=False)
