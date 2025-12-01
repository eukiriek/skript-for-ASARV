# Приведение поля "Норма" к единому формату HH:MM:SS

# 1. Преобразуем любые варианты формата (ч:мм:сс, чч:мм:сс, пробелы, точки)
norma_clean = (
    df["Норма"]
    .astype(str)
    .str.strip()
    .str.replace('.', ':', regex=False)
)

# 2. Добавляем ведущий ноль, если часы заданы одной цифрой
#   Пример: "7:30:00" → "07:30:00"
norma_clean = norma_clean.str.replace(
    r'^(\d):',  
    r'0\1:', 
    regex=True
)

# 3. Преобразуем в timedelta → обратно в строку HH:MM:SS
df["Норма"] = pd.to_timedelta(norma_clean, errors="coerce").dt.components.apply(
    lambda x: f"{int(x['hours']):02d}:{int(x['minutes']):02d}:{int(x['seconds']):02d}",
    axis=1
)
