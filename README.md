df_main = df_main.merge(
    df_add[['ФИО', 'Поле1']],
    on='ФИО',
    how='left'
)

df_main = df_main.merge(
    df_add[['ФИО2', 'Поле1']],           # подтягиваем только нужное
    left_on='ФИО',                       # ключ в основной таблице
    right_on='ФИО2',                     # ключ в дополнительной
    how='left'
)
