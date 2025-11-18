# --удаление exceptions_sp.
df_ref = pd.read_excel("spravochnik.xlsx", sheet_name='exceptions_sp')  # справочник
df['Полное наименование'] = df['Полное наименование'].astype(str).str.strip()
df_ref['exception'] = df_ref['exception'].astype(str).str.strip()

match_values = set(df_ref['exception'].unique())

# --- новые строки: сначала выбираем удаляемые строки ---
df_exceptions_removed = df[df['Полное наименование'].isin(match_values)].copy()

# --- затем удаляем из основного df ---
df = df[~df['Полное наименование'].isin(match_values)].copy()

# --- сохраняем исключённые строки в новый файл ---
df_exceptions_removed.to_excel("exception.xlsx", index=False)

print('-- удаление exceptions_sp.')
print(f'Сохранено удалённых строк: {len(df_exceptions_removed)}')
