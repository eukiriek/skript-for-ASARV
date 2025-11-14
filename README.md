df["Новое отработанное время"] = pd.to_timedelta(df["Новое отработанное время"], errors="coerce")
df["Новая норма"] = pd.to_timedelta(df["Новая норма"], errors="coerce")

df["Новые переработки"] = df["Новое отработанное время"] - df["Новая норма"]



print(df["Новое отработанное время"].dtype)
print(df["Новая норма"].dtype)


# Функция для перевода timedelta в формат ЧЧ:ММ:СС
def td_to_hms(td):
    if pd.isna(td):
        return None
    # td может быть не timedelta — защищаемся
    try:
        total_seconds = int(td.total_seconds())
    except:
        return None

    # преобразуем в HH:MM:SS
    hours = total_seconds // 3600
    minutes = (total_seconds % 3600) // 60
    seconds = total_seconds % 60

    return f"{hours:02d}:{minutes:02d}:{seconds:02d}"


df["новое_поле"] = df["timedelta_поле"].apply(td_to_hms)
