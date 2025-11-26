df_main = df_main.merge(
    df_add[['ФИО', 'Поле1']],
    on='ФИО',
    how='left'
)
