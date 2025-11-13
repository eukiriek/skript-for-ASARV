import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем колонку M в datetime-формат (только время)
df["M"] = pd.to_datetime(df["M"], format="%H:%M:%S", errors="coerce")

# Округление ко ближайшему часу
df["N"] = (df["M"] + pd.Timedelta(minutes=30)).dt.floor("H")

# Приводим к формату HH:MM:SS
df["N"] = df["N"].dt.strftime("%H:%M:%S")

df.to_excel("your_file_updated.xlsx", index=False)

print("Готово! Поле N создано и время округлено до ближайшего часа.")
