import pandas as pd

# ==============================
# Вспомогательные функции
# ==============================

def td_to_hms(td):
    """
    функция замены формата timedelta64 на hh:mm:ss
    """
    if pd.isna(td):
        return None  # td может быть не timedelta — защищаемся
    try:
        total_seconds = int(td.total_seconds())
    except Exception:
        return None

    hours = total_seconds // 3600
    minutes = (total_seconds % 3600) // 60
    seconds = total_seconds % 60

    return f"{hours:02d}:{minutes:02d}:{seconds:02d}"


def clean_str(series):
    """Универсальная очистка строковых полей: приведение к str и trim"""
    return series.astype(str).str.strip()


def drop_empty_time_rows(df, cols):
    """
    Удаление строк, если в указанных полях пустые значения (пустая строка / пробелы / NaN).
    """
    df[cols] = df[cols].replace(r'^\s*$', pd.NA, regex=True)
    df = df.dropna(subset=cols)
    return df


def drop_zero_time_rows(df, cols):
    """
    Удаление строк, если время в указанных полях равно 00:00:00.
    """
    for col in cols:
        t = pd.to_datetime(
            df[col].astype(str).str.strip().str.replace('.', ':', regex=False),
            format="%H:%M:%S",
            errors="coerce"
        )
        mask_midnight = (t.dt.hour == 0) & (t.dt.minute == 0) & (t.dt.second == 0)
        df = df.loc[~mask_midnight].copy()
    return df


def apply_exceptions(df, sheet_name, target_col, sprav_file="spravochnik.xlsx"):
    """
    Удаление строк по справочнику исключений.
    Удалённые строки сохраняются в exception.xlsx (как и в исходном коде — последняя перезаписывает файл).
    """
    df_ref = pd.read_excel(sprav_file, sheet_name=sheet_name)
    df[target_col] = clean_str(df[target_col])
    df_ref["exception"] = clean_str(df_ref["exception"])

    match_values = set(df_ref["exception"].unique())
    mask = df[target_col].isin(match_values)

    df_exceptions_removed = df[mask].copy()
    df_exceptions_removed.to_excel("exception.xlsx", index=False)

    df = df[~mask].copy()
    return df


# ==============================
# Часть 1: Работа с файлом ASARV_NEW
# ==============================

print('ЧАСТЬ 1: РАБОТА С ФАЙЛОМ ASARV_NEW')
print("Преобразование данных")

# --- Загрузка и базовая очистка
df = pd.read_excel("asarv.xlsx")
df.columns = df.columns.str.replace(r'[\r\n]+', '', regex=True).str.strip()

# заполнение пустых ячеек в полях
cols_ffill = ['Подразделение', 'Сотрудник', 'Имя', 'Должность']
df[cols_ffill] = df[cols_ffill].ffill()
print('-- заполнение пустых ячеек в полях.')

# ==============================
# добавление полного названия СП
# ==============================
df_ref_sp = pd.read_excel("spravochnik.xlsx", sheet_name='SP')
df['Подразделение'] = clean_str(df['Подразделение'])
df_ref_sp['short_name'] = clean_str(df_ref_sp['short_name'])

sp_map = df_ref_sp.set_index('short_name')['full_name']
df['Полное наименование'] = df['Подразделение'].map(sp_map)
print('-- добавление полного названия СП.')

# ==============================
# добавление СП2 и СП3
# ==============================
df_ref = pd.read_excel("shtatka.xlsx")

# Ключевые поля для сопоставления
df["Сотрудник"] = clean_str(df["Сотрудник"])
df_ref["ТабНомер"] = clean_str(df_ref["ТабНомер"])

df["Календарный день"] = pd.to_datetime(df["Календарный день"], errors="coerce", dayfirst=True)
df_ref["ДатаНазнНаДолжность"] = pd.to_datetime(df_ref["ДатаНазнНаДолжность"], errors="coerce", dayfirst=True)
df_ref["ДатаУвольнения"] = pd.to_datetime(df_ref["ДатаУвольнения"], errors="coerce", dayfirst=True)

# Если "Дата увольнения" пустая, считаем, что сотрудник всё ещё работает
open_end_date = pd.Timestamp("2099-12-31")
df_ref["ДатаУвольнения"] = df_ref["ДатаУвольнения"].fillna(open_end_date)

# Подготовка для сопоставления
df = df.reset_index().rename(columns={"index": "row_id"})
ref_keep = ["ТабНомер", "ДатаНазнНаДолжность", "ДатаУвольнения",
            "Подразд-е уровень 2", "Подразд-е уровень 3"]
df_ref = df_ref[ref_keep].copy()

merged = df.merge(
    df_ref,
    how="left",
    left_on="Сотрудник",
    right_on="ТабНомер",
    suffixes=("", "_ref")
)

in_range = (
    (merged["Календарный день"] >= merged["ДатаНазнНаДолжность"]) &
    (merged["Календарный день"] <= merged["ДатаУвольнения"])
)
candidates = merged[in_range].copy()
candidates = candidates.sort_values(["row_id", "ДатаНазнНаДолжность"])
best_match = candidates.drop_duplicates(subset="row_id", keep="last")

df = df.set_index("row_id").copy()
df["Подразд-е уровень 2"] = pd.NA
df["Подразд-е уровень 3"] = pd.NA

df.loc[
    best_match["row_id"].values,
    ["Подразд-е уровень 2", "Подразд-е уровень 3"]
] = best_match.set_index("row_id")[["Подразд-е уровень 2", "Подразд-е уровень 3"]].values

df = df.reset_index(drop=True)
print('-- добавление СП2 и СП3.')

# ==============================
# удаление строк с пустым временем
# ==============================
time_cols = ['Время входа', 'Время выхода', 'Норма', 'Отработанное время']
df = drop_empty_time_rows(df, time_cols)
print('-- удаление строк если время входа, выхода, норма, отр. время - пустое.')

# ==============================
# удаление строк с нулевым временем
# ==============================
df = drop_zero_time_rows(df, time_cols)
print('-- удаление строк если время входа, выхода, норма, отр. время - нулевое.')

# ==============================
# удаление строк с отрицательным временем (как в исходном коде,
# фактически таких значений не будет, но блок оставлен для совместимости)
# ==============================
for col in ["Время входа", "Время выхода", "Норма", "Отработанное время"]:
    t = pd.to_datetime(
        df[col].astype(str).str.strip().str.replace('.', ':', regex=False),
        format="%H:%M:%S",
        errors="coerce"
    )
    # отрицательных часов у времени суток не будет, но условие сохраняем
    mask_neg = (t.dt.hour < 0) & (t.dt.minute < 0) & (t.dt.second < 0)
    df = df.loc[~mask_neg].copy()
print('-- удаление строк если время входа, выхода, норма, отр. время - отрицательное.')

# ==============================
# удаление exceptions_ID, exceptions_roles, exceptions_sp
# ==============================
df = apply_exceptions(df, sheet_name='exceptions_ID', target_col='Сотрудник')
print('-- удаление exceptions_ID и сохранение в новом файле.')

df = apply_exceptions(df, sheet_name='exceptions_roles', target_col='Должность')
print('-- удаление exceptions_roles и сохранение в новом файле.')

df = apply_exceptions(df, sheet_name='exceptions_sp', target_col='Полное наименование')
print('-- удаление exceptions_sp и сохранение в новом файле.')

# ==============================
# добавление признака вн.совместитель
# ==============================
df_ref = pd.read_excel("spravochnik.xlsx", sheet_name='sovmestitely')
df['Сотрудник'] = clean_str(df['Сотрудник'])
df_ref['ТабНомер'] = clean_str(df_ref['ТабНомер'])
df['Флаг совместитель'] = df['Сотрудник'].isin(df_ref['ТабНомер']).astype(int)
print('-- добавление признака вн.совместитель.')

# ==============================
# добавление признака СУРВ
# ==============================
df_ref = pd.read_excel("spravochnik.xlsx", sheet_name='SURV')
df['Сотрудник'] = clean_str(df['Сотрудник'])
df_ref['ТабНомер'] = clean_str(df_ref['ТабНомер'])
df['Флаг СУРВ'] = df['Сотрудник'].isin(df_ref['ТабНомер']).astype(int)
print('-- добавление признака СУРВ.')

# ==============================
# добавление признака категория роли
# ==============================
df_ref = pd.read_excel("spravochnik.xlsx", sheet_name='category')
df_ref['role'] = clean_str(df_ref['role'])
df['Должность'] = clean_str(df['Должность'])

role_map = df_ref.set_index('role')['category']
df['Категория должности'] = df['Должность'].map(role_map)
print('-- добавление признака категории должности.')

# ==============================
# расчет флага праздника и флага "норма < 8"
# ==============================
df_ref = pd.read_excel("spravochnik.xlsx", sheet_name='spec_date')
df_ref["date-1"] = pd.to_datetime(df_ref["date-1"], errors="coerce")
holiday_set = set(df_ref["date-1"].dropna().unique())

df["Флаг праздник"] = df["Календарный день"].isin(holiday_set).astype(int)

norma_str = df['Норма'].astype(str).str.strip()
time_parts = norma_str.str.extract(r'^\s*(\d{1,2}):(\d{1,2}):(\d{1,2})\s*$')
time_parts = time_parts.fillna(0).astype(int)

hours = time_parts[0]
minutes = time_parts[1]
seconds = time_parts[2]
total_seconds = hours * 3600 + minutes * 60 + seconds

df['Флаг норма<8'] = (total_seconds < 8 * 3600).astype(int)
print('-- добавление признака Флаг норма < 8.')

# ==============================
# добавление признака ЕСЦ
# ==============================
df["Полное наименование"] = clean_str(df["Полное наименование"])
df["ЕСЦ"] = df["Полное наименование"].str.contains("сервисный центр", case=False, na=False).astype(int)
print('-- добавление признака ЕСЦ.')

# ==============================
# добавление признака КЦ (используем финальную логику: только по уровню 2)
# ==============================
df["Подразд-е уровень 2"] = clean_str(df["Подразд-е уровень 2"])
df["КЦ"] = df["Подразд-е уровень 2"].str.contains("контакт-центр", case=False, na=False).astype(int)
print('-- добавление признака КЦ.')

# ==============================
# добавление поля новое время прихода/выхода
# ==============================
df["Время входа"] = pd.to_datetime(df["Время входа"], format="%H:%M:%S", errors="coerce")
df["Время выхода"] = pd.to_datetime(df["Время выхода"], format="%H:%M:%S", errors="coerce")

df["Новое время входа"] = (df["Время входа"] + pd.Timedelta(minutes=30)).dt.floor("H")
df["Новое время выхода"] = (df["Время выхода"] + pd.Timedelta(minutes=30)).dt.floor("H")

df["Новое время входа"] = df["Новое время входа"].dt.strftime("%H:%M:%S")
df["Новое время выхода"] = df["Новое время выхода"].dt.strftime("%H:%M:%S")
print("-- добавление поля новое время прихода, новое время выхода")

# ==============================
# создание новых полей: Новая норма, День недели
# ==============================
df['Новая норма'] = pd.to_datetime(df['Норма'], errors='coerce').dt.strftime('%H:%M:%S')

df['День недели'] = pd.to_datetime(df['Календарный день'], errors='coerce').dt.day_name()

weekday_map = {
    'Monday': 'Понедельник',
    'Tuesday': 'Вторник',
    'Wednesday': 'Среда',
    'Thursday': 'Четверг',
    'Friday': 'Пятница',
    'Saturday': 'Суббота',
    'Sunday': 'Воскресенье',
}
df['День недели'] = df['День недели'].map(weekday_map)
print('-- создание новых полей.')

# ==============================
# изменение новой нормы в пн-чт
# ==============================
df['Новая норма'] = pd.to_datetime(df['Новая норма'], errors='coerce')
df['Флаг СУРВ'] = pd.to_numeric(df['Флаг СУРВ'], errors='coerce')

weekdays_mon_to_thu = ['Понедельник', 'Вторник', 'Среда', 'Четверг']
mask_01 = (
    df['День недели'].isin(weekdays_mon_to_thu) &
    (df['Флаг СУРВ'] == 0) &
    (df['Флаг норма<8'] == 0)
)

df.loc[mask_01, 'Новая норма'] = (
    df.loc[mask_01, 'Новая норма'].dt.normalize() +
    pd.Timedelta(hours=8, minutes=15, seconds=0)
)
df['Новая норма'] = df['Новая норма'].dt.strftime('%H:%M:%S')
print('-- изменение новой нормы в пн-чт, сохранение нормы по СУРВ и нормы < 8.')

# ==============================
# изменение новой нормы в пт
# ==============================
df['Новая норма'] = pd.to_datetime(df['Новая норма'], errors='coerce')
df['Флаг СУРВ'] = pd.to_numeric(df['Флаг СУРВ'], errors='coerce')

weekdays_friday = ['Пятница']
mask_02 = (
    df['День недели'].isin(weekdays_friday) &
    (df['Флаг СУРВ'] == 0) &
    (df['Флаг норма<8'] == 0)
)

df.loc[mask_02, 'Новая норма'] = (
    df.loc[mask_02, 'Новая норма'].dt.normalize() +
    pd.Timedelta(hours=6, minutes=45, seconds=0)
)
df['Новая норма'] = df['Новая норма'].dt.strftime('%H:%M:%S')
print('-- изменение новой нормы в пт, сохранение нормы по СУРВ и нормы < 8.')

# ==============================
# добавление поля Время обеда
# ==============================
df["Время обеда"] = pd.NA
mask_obed_empty = df['Время обеда'].isna() | (df['Время обеда'].astype(str).str.strip() == '')
time_M = pd.to_timedelta(df['Новая норма'].astype(str), errors='coerce')

# Если время пусто и СУРВ, то 1 час
mask_obed1 = mask_obed_empty & (df['Флаг СУРВ'] == 1)
df.loc[mask_obed1, 'Время обеда'] = '01:00:00'
mask_obed_empty = df['Время обеда'].isna() | (df['Время обеда'].astype(str).str.strip() == '')

# Если время пусто и норма 8.15, то 45 мин
mask_obed2 = mask_obed_empty & (time_M == pd.Timedelta(hours=8, minutes=15))
df.loc[mask_obed2, 'Время обеда'] = '00:45:00'
mask_obed_empty = df['Время обеда'].isna() | (df['Время обеда'].astype(str).str.strip() == '')

# Если время пусто и норма 6.45, то 45 мин
mask_obed3 = mask_obed_empty & (time_M == pd.Timedelta(hours=6, minutes=45))
df.loc[mask_obed3, 'Время обеда'] = '00:45:00'
mask_obed_empty = df['Время обеда'].isna() | (df['Время обеда'].astype(str).str.strip() == '')

# Если время пусто и норма между 5.01 и 5.44, то 1 час
mask_obed4 = (
    mask_obed_empty &
    (time_M >= pd.Timedelta(hours=5, minutes=1)) &
    (time_M <= pd.Timedelta(hours=5, minutes=44))
)
df.loc[mask_obed4, 'Время обеда'] = '01:00:00'
mask_obed_empty = df['Время обеда'].isna() | (df['Время обеда'].astype(str).str.strip() == '')

# Если время пусто и норма между 6.46 и 8.14, то 1 час
mask_obed5 = (
    mask_obed_empty &
    (time_M >= pd.Timedelta(hours=6, minutes=46)) &
    (time_M <= pd.Timedelta(hours=8, minutes=14))
)
df.loc[mask_obed5, 'Время обеда'] = '01:00:00'
mask_obed_empty = df['Время обеда'].isna() | (df['Время обеда'].astype(str).str.strip() == '')

# Для остальных 0 часов
df.loc[mask_obed_empty, 'Время обеда'] = '00:00:00'
print('-- добавление поле время обеда.')

print("Расчеты итоговых полей")

# ==============================
# расчет нового отработанного времени
# ==============================
df["Время входа"] = pd.to_datetime(df["Время входа"], errors="coerce")
df["Время выхода"] = pd.to_datetime(df["Время выхода"], errors="coerce")

for col in ["Отработанное время", "Время обеда", "Время отсутствия"]:
    df[col] = pd.to_timedelta(df[col].astype(str), errors="coerce")

df["Новое отработанное время"] = pd.NaT

# Условие 1: если отработанное время >= 24 часов, просто копируем
mask_ge_24 = df["Отработанное время"] >= pd.Timedelta(hours=24)
df.loc[mask_ge_24, "Новое отработанное время"] = df.loc[mask_ge_24, "Отработанное время"]

# Условие 2: иначе считаем
mask_other = ~mask_ge_24
max_break = df[["Время обеда", "Время отсутствия"]].max(axis=1)
df.loc[mask_other, "Новое отработанное время"] = (
    df.loc[mask_other, "Время выхода"] -
    df.loc[mask_other, "Время входа"] -
    max_break[mask_other]
)

# Для длительностей -> строка hh:mm:ss
for col in ["Отработанное время", "Время обеда", "Время отсутствия", "Новое отработанное время"]:
    df[col] = df[col].apply(td_to_hms)

# Для времени входа/выхода берём только время из datetime
for col in ["Время входа", "Время выхода"]:
    df[col] = pd.to_datetime(df[col], errors="coerce").dt.strftime("%H:%M:%S")
print('-- расчет поля новое отработанное время.')

# ==============================
# расчет новой нормы с учетом праздников
# (переводим НОВУЮ НОРМУ в timedelta перед вычитанием часа)
# ==============================
df["Новая норма"] = pd.to_timedelta(df["Новая норма"], errors="coerce")
mask_hol = (df["Флаг праздник"] == 1) & (df["Флаг СУРВ"] == 0)
df.loc[mask_hol, "Новая норма"] = df.loc[mask_hol, "Новая норма"] - pd.Timedelta(hours=1)
print('-- расчет новой нормы с учетом праздников.')

# ==============================
# расчет поля новые переработки
# ==============================
df["Новое отработанное время"] = pd.to_timedelta(df["Новое отработанное время"], errors="coerce")
# Новая норма уже в timedelta
df["Новые переработки"] = df["Новое отработанное время"] - df["Новая норма"]

df.loc[
    (df["Новые переработки"].dt.total_seconds() <= 0) |
    (df["Новые переработки"].isna()),
    "Новые переработки"
] = pd.Timedelta(0)
print('-- расчет поля новые переработки.')

# ==============================
# расчет поля новые недоработки
# ==============================
df["Отработанное время"] = pd.to_timedelta(df["Отработанное время"].astype(str), errors="coerce")
df["Новые недоработки"] = df["Новая норма"] - df["Отработанное время"]

df.loc[
    (df["Новые недоработки"].dt.total_seconds() <= 0) |
    (df["Новые недоработки"].isna()),
    "Новые недоработки"
] = pd.Timedelta(0)
print('-- расчет поля новые недоработки.')

# ==============================
# Изменение формата полей на чч:мм:сс перед выгрузкой
# ==============================
df["Отработанное время"] = df["Отработанное время"].apply(td_to_hms)
df["Новая норма"] = df["Новая норма"].apply(td_to_hms)
df["Новые недоработки"] = df["Новые недоработки"].apply(td_to_hms)
df["Новое отработанное время"] = df["Новое отработанное время"].apply(td_to_hms)
print('-- корректировка формата полей перед выгрузкой.')

# ==============================
# создание листа с агрегациями
# ==============================
df["Новые переработки"] = pd.to_timedelta(df["Новые переработки"])
agg = df.groupby("Имя", as_index=False)["Новые переработки"].sum()

df["Новые недоработки"] = df["Новые недоработки"].apply(td_to_hms)
df["Новые переработки"] = df["Новые переработки"].apply(td_to_hms)
print('-- создание листа с агрегацией.')

# ==============================
# Выгрузка
# ==============================
df.to_excel("exceptions.xlsx", index=False)
