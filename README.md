import re
import unicodedata
import pandas as pd

def clean_columns(cols):
    cleaned = []
    used = {}

    for c in cols:
        c = "" if c is None else str(c)

        # 1) нормализация юникода (убирает “умные” символы и т.п.)
        c = unicodedata.normalize("NFKC", c)

        # 2) обрезаем пробелы/табуляции/переносы
        c = c.strip()

        # 3) заменяем переносы и табы на пробел
        c = re.sub(r"[\r\n\t]+", " ", c)

        # 4) убираем "лишние" пробелы
        c = re.sub(r"\s+", " ", c)

        # 5) делаем техничное имя: всё, что не буква/цифра/_ -> "_"
        # (оставляем кириллицу тоже, это нормально для pandas)
        c = re.sub(r"[^\w]+", "_", c, flags=re.UNICODE)

        # 6) схлопываем "__" и убираем "_" по краям
        c = re.sub(r"_+", "_", c).strip("_")

        # 7) если имя пустое или типа Unnamed
        if not c or c.lower().startswith("unnamed"):
            c = "col"

        # 8) если начинается с цифры — добавим префикс
        if c[0].isdigit():
            c = f"col_{c}"

        # 9) обеспечиваем уникальность (col, col_2, col_3...)
        base = c
        n = used.get(base, 0) + 1
        used[base] = n
        if n > 1:
            c = f"{base}_{n}"

        cleaned.append(c)

    return cleaned

# === пример использования ===
in_path = "input.xlsx"
out_path = "output_cleaned.xlsx"

df = pd.read_excel(in_path)              # при необходимости: sheet_name="Лист1"
df.columns = clean_columns(df.columns)

df.to_excel(out_path, index=False)
print("Готово:", out_path)
