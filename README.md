import pandas as pd

# 1. Преобразуем "дата_приема" из object в datetime
df["дата_приема"] = pd.to_datetime(
    df["дата_приема"],
    dayfirst=True,      # формат dd.mm.yyyy (31.12.2023 00:00:00)
    errors="coerce"     # некорректные значения станут NaT
)

# 2. Берём сегодняшнюю дату (без времени)
today = pd.Timestamp.today().normalize()

# 3. Считаем разницу в днях: сегодня - дата_приема
df["разница_дней"] = (today - df["дата_приема"]).dt.days
