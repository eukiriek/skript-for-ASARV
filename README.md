import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Преобразуем Z в формат времени (даже если это число)
df['Z'] = pd.to_timedelta(df['Z'], errors='coerce')

# Удаляем строки, где время ровно 00:00:00 или значение не распознаётся
df = df[df['Z'] != pd.Timedelta(0)]

df.to_excel("your_file_updated.xlsx", index=False)
