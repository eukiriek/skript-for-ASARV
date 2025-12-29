import numpy as np

# --- нормализация табельных номеров во всех таблицах ---
df["Таб Номер"] = normalize_tab(df["Таб Номер"])
df_otpusk["Табельный номер"] = normalize_tab(df_otpusk["Табельный номер"])
df_sprav["Табельный номер"] = normalize_tab(df_sprav["Табельный номер"])

# --- мапы по otpusk и spravochnik ---

# otpusk: суммируем поле "Всего" по табельному номеру
otpusk_map = (
    df_otpusk
    .groupby("Табельный номер", as_index=True)["Всего"]
    .sum()
)

# spravochnik: считаем доп. дни (ДопДни + Северные) и суммируем по табельному
df_sprav[["ДопДни", "Северные"]] = df_sprav[["ДопДни", "Северные"]].fillna(0)
df_sprav["доп_все"] = df_sprav["ДопДни"] + df_sprav["Северные"]

sprav_map = (
    df_sprav
    .groupby("Табельный номер", as_index=True)["доп_все"]
    .sum()
)

# --- расчёт накопленных дней отпуска ---

df["Накопленные дни отпуска"] = 0

mask_180_730 = (df["Стаж, дни"] > 180) & (df["Стаж, дни"] < 731)
mask_731_plus = df["Стаж, дни"] >= 731

# 1) стаж 180–730 дней → просто переносим "Всего" из otpusk.xlsx
df.loc[mask_180_730, "Накопленные дни отпуска"] = (
    df.loc[mask_180_730, "Таб Номер"].map(otpusk_map)
)

# 2) стаж 731+ дней:
#    базовая формула: (24 + ДопОтпуск + Северные) * 2
#    но если она даёт значение больше, чем "Всего" из otpusk.xlsx,
#    то берём значение из otpusk.xlsx

tab_731 = df.loc[mask_731_plus, "Таб Номер"]

# формула (24 + ДопОтпуск + Северные) * 2
calc_days = (24 + tab_731.map(sprav_map)) * 2

# значение "Всего" из otpusk.xlsx
otpusk_days = tab_731.map(otpusk_map)

# результат с учётом условия "не больше, чем otpusk"
res_731 = calc_days.copy()
has_otp = otpusk_days.notna()

res_731[has_otp] = np.where(
    calc_days[has_otp] > otpusk_days[has_otp],
    otpusk_days[has_otp],      # если формула даёт больше — берём otpusk.xlsx
    calc_days[has_otp],        # иначе оставляем формулу
)

df.loc[mask_731_plus, "Накопленные дни отпуска"] = res_731

# где ничего не нашлось — ставим 0
df["Накопленные дни отпуска"] = df["Накопленные дни отпуска"].fillna(0)
