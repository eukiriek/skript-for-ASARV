# переводим переработки в timedelta
df["Новые переработки"] = pd.to_timedelta(df["Новые переработки"])

# считаем агрегацию по имени
agg = df.groupby("Имя", as_index=False)["Новые переработки"].sum()

# функция перевода timedelta -> "чч:мм:сс"
def td_to_hms(td):
    if pd.isna(td):
        return None
    total_seconds = int(td.total_seconds())
    hours   = total_seconds // 3600
    minutes = (total_seconds % 3600) // 60
    seconds = total_seconds % 60
    return f"{hours:02d}:{minutes:02d}:{seconds:02d}"

# переводим в формат "25:00:00" и т.п.
agg["Новые переработки"] = agg["Новые переработки"].apply(td_to_hms)

# при желании можно конвертировать и в основном df
df["Новые переработки"] = df["Новые переработки"].apply(td_to_hms)

# выгрузка
with pd.ExcelWriter("asarv_new.xlsx") as writer:
    df.to_excel(writer, sheet_name="asarv_new", index=False)
    agg.to_excel(writer, sheet_name="агрегация", index=False)
