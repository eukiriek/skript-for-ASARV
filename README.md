tab_731 = df.loc[mask_731_plus, "Таб Номер"]

# формула (24 + ДопОтпуск + Северные) * 2
calc_days = (24 + tab_731.map(sprav_map)) * 2

# значение "Всего" из otpusk.xlsx
otpusk_days = tab_731.map(otpusk_map)

# результат с учётом условия "не больше, чем otpusk"
res_731 = calc_days.copy()
has_otp = otpusk_days.notna()

# приводим оба ряда к числам, чтобы сравнение было корректным
calc_num = pd.to_numeric(calc_days[has_otp], errors="coerce")
otp_num  = pd.to_numeric(otpusk_days[has_otp], errors="coerce")

# если формула даёт больше, чем otpusk → берём otpusk
res_731[has_otp] = calc_num.where(calc_num <= otp_num, otp_num)

df.loc[mask_731_plus, "Накопленные дни отпуска"] = res_731
