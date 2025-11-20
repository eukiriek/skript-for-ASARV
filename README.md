import pandas as pd

# 1. Загружаем файл, откуда берем значения a и b
# пример: файл data.xlsx с колонками a и b
source = pd.read_excel("data.xlsx")

# 2. Создаем пустую таблицу с нужными полями
df = pd.DataFrame(columns=["a", "b", "c"])

# 3. Заполняем поля a и b значениями из файла
df["a"] = source["a"]
df["b"] = source["b"]

# 4. Проверяем уникальность поле a
if df["a"].is_unique:
    print("Все значения 'a' уникальные")
else:
    dup = df[df["a"].duplicated(keep=False)]
    raise ValueError(f"Найдены неуникальные значения в 'a':\n{dup}")

# 5. Выгрузка в Excel
df.to_excel("new_table.xlsx", index=False)

print("Готово! Файл new_table.xlsx создан.")
