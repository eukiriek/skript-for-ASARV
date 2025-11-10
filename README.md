import pandas as pd

df = pd.read_excel("your_file.xlsx")

# Создаем новый столбец и копируем туда значения из M
df['NewColumn'] = df['M']

df.to_excel("your_file_with_new_column.xlsx", index=False)
