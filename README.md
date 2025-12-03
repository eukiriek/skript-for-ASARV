"""
Прогноз временного ряда методом Хольта–Винтерса (ExponentialSmoothing)

Ожидаемые поля в test.xlsx:
- date    : дата наблюдения
- meaning : значение метрики
"""

import warnings
import numpy as np
import pandas as pd

from statsmodels.tsa.holtwinters import ExponentialSmoothing
from sklearn.metrics import mean_absolute_error

warnings.filterwarnings("ignore")


# ============================
# Загрузка и подготовка данных
# ============================

# Загружаем файл с данными
df = pd.read_excel("test.xlsx")

# Приводим даты к datetime (день первый в формате dd.mm.yyyy)
df["date"] = pd.to_datetime(df["date"], dayfirst=True, errors="coerce")

# Приводим значения метрики к числовому типу
df["meaning"] = pd.to_numeric(df["meaning"], errors="coerce")

# Удаляем строки с пропущенными датами/значениями, сортируем по дате и делаем индексом
df = (
    df.dropna(subset=["date", "meaning"])
      .sort_values("date")
      .set_index("date")
)

if len(df) < 24:
    raise ValueError(f"Недостаточно наблюдений: {len(df)}. Нужен минимум 24.")

series = df["meaning"]

# Если у вас месячные данные — можно явно задать частоту 'MS' (Month Start)
# series = series.asfreq("MS")


# ============================
# Train / Test сплит
# ============================

FORECAST_HORIZON = 12  # сколько периодов хотим прогнозировать
test_steps = min(FORECAST_HORIZON, max(1, len(series) // 4))

train = series.iloc[:-test_steps]
test = series.iloc[-test_steps:]

print(f"Всего наблюдений: {len(series)}; train: {len(train)}, test: {len(test)}")


# ============================
# Метрика MAPE (дополнительно к MAE)
# ============================

def mape(y_true, y_pred) -> float:
    """MAPE с защитой от деления на ноль (в %)."""
    y_true = np.array(y_true, dtype=float)
    y_pred = np.array(y_pred, dtype=float)
    eps = 1e-9
    denom = np.clip(np.abs(y_true), eps, None)
    return float(np.mean(np.abs((y_true - y_pred) / denom)) * 100)


# ============================
# Модель Хольта–Винтерса
# ============================

"""
Ключевые параметры метода Хольта–Винтерса (ExponentialSmoothing):

trend:
    - None      : тренд не моделируется
    - 'add'     : аддитивный тренд (линейный рост/падение; добавляется к уровню)
    - 'mul'     : мультипликативный тренд (изменение в процентах; умножается на уровень)

damped_trend:
    - False     : тренд не «гасится» (может расти/падать линейно далеко в будущее)
    - True      : затухающая версия тренда, рост/падение замедляется по мере движения в будущее

seasonal:
    - None      : сезонность не моделируется
    - 'add'     : аддитивная сезонность (амплитуда сезонности примерно постоянна)
    - 'mul'     : мультипликативная сезонность (амплитуда растёт вместе с уровнем ряда)

seasonal_periods:
    - длина сезонного цикла:
        * 12  : месячные данные с годовой сезонностью
        * 7   : ежедневные данные с недельной сезонностью
        * 4   : квартальные данные с годовой сезонностью (4 квартала)

use_boxcox:
    - False     : без преобразования
    - True      : применить Box-Cox трансформацию к данным (помогает при сильной нестационарности по дисперсии)
    - 'log'     : логарифмическое преобразование (частный случай Box-Cox с λ = 0)

initialization_method:
    - 'estimated' : начальные значения уровня/тренда/сезонности оцениваются из данных (по умолчанию)
    - 'heuristic' : быстрые эвристики (можно использовать для больших рядов)
    - 'legacy-heuristic' : старый подход из statsmodels (для совместимости)
"""

# Настройки модели Хольта–Винтерса
TREND = "add"            # тип тренда: None / 'add' / 'mul'
DAMPED = False           # затухающий тренд: True / False
SEASONAL = "add"         # тип сезонности: None / 'add' / 'mul'
SEASONAL_PERIODS = 12    # длина сезонного цикла (для месячных данных = 12)
USE_BOXCOX = False       # преобразование Box-Cox: False / True / 'log'

# Создаём модель на обучающей выборке
hw_model = ExponentialSmoothing(
    train,
    trend=TREND,
    damped_trend=DAMPED,
    seasonal=SEASONAL,
    seasonal_periods=SEASONAL_PERIODS,
    initialization_method="estimated",
    use_boxcox=USE_BOXCOX,
)

# Обучаем модель: alpha, beta, gamma и другие параметры подбираются автоматически по минимуму SSE
hw_fit = hw_model.fit(optimized=True)

# Прогноз на тестовый горизонт (для оценки качества)
hw_fc_test = hw_fit.forecast(steps=test_steps)

# Метрики на тесте
hw_mae = mean_absolute_error(test, hw_fc_test)
hw_mape = mape(test, hw_fc_test)

print("\n=== Holt–Winters (ExponentialSmoothing) ===")
print(f"Параметры модели: trend={TREND}, damped_trend={DAMPED}, "
      f"seasonal={SEASONAL}, seasonal_periods={SEASONAL_PERIODS}, use_boxcox={USE_BOXCOX}")
print(f"MAE  (test): {hw_mae:.4f}")
print(f"MAPE (test): {hw_mape:.2f}%")

# Итоговый прогноз на FORECAST_HORIZON шагов от конца всего ряда
hw_fit_full = ExponentialSmoothing(
    series,
    trend=TREND,
    damped_trend=DAMPED,
    seasonal=SEASONAL,
    seasonal_periods=SEASONAL_PERIODS,
    initialization_method="estimated",
    use_boxcox=USE_BOXCOX,
).fit(optimized=True)

hw_fc_full = hw_fit_full.forecast(steps=FORECAST_HORIZON)

print(f"\nПрогноз Хольта–Винтерса на {FORECAST_HORIZON} периодов:")
print(hw_fc_full.round(4))
