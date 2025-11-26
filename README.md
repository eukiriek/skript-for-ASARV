import pandas as pd

# предположим, что у обеих таблиц ключ называется "id"
# и текстовые поля: "ФИО" в основной и "Сотрудник" в дополнительной

# соединяем таблицы по ключу
df = df_main.merge(
    df_add[['id', 'Сотрудник']], 
    on='id',
    how='left',
    suffixes=('', '_add')
)

# создаём новое поле там, где значения ФИО и Сотрудник не совпадают
df['ФИО_из_доп'] = df.apply(
    lambda row: row['Сотрудник'] if row['ФИО'] != row['Сотрудник'] else None,
    axis=1
)

# если нужно — сохранить обратно в основную таблицу
df_main = df.copy()




df_main['ФИО'] = df_main['ФИО'].str.strip().str.lower()
df_add['ФИО']  = df_add['ФИО'].str.strip().str.lower()

df_main = df_main.merge(
    df_add[['ФИО', 'Поле1', 'Поле2']], 
    on='ФИО',
    how='left'
)
