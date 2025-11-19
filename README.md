import warnings
import itertools

import pandas as pd
import matplotlib.pyplot as plt
from statsmodels.tsa.arima.model import ARIMA

# ----- 1. Подготовка ряда -----

# meaning — твой исходный датафрейм

df = meaning.copy()

# если есть колонка date — делаем её индексом
if 'date' in df.columns:
    df['date'] = pd.to_datetime(df['date'])
    df.set_index('date', inplace=True)

# имя колонки со значением, поправь если нужно
value_col = 'value'

# оставляем только ряд
ts = df[value_col].astype(float)

# приводим индекс к datetime
ts.index = pd.to_datetime(ts.index)

# пытаемся восстановить частоту
freq = pd.infer_freq(ts.index)
if freq is not None:
    ts = ts.asfreq(freq)

# при необходимости можно залить пропуски
ts = ts.fillna(method='ffill')

# ----- 2. Автоподбор (p, d, q) по AIC -----

warnings.filterwarnings("ignore")

p = d = q = range(0, 3)  # диапазоны можно расширить
pdq = list(itertools.product(p, d, q))

best_aic = float("inf")
best_order = None
best_model = None

for order in pdq:
    try:
        model = ARIMA(ts, order=order)
        results = model.fit()
        if results.aic < best_aic:
            best_aic = results.aic
            best_order = order
            best_model = results
    except Exception:
        # некоторые комбинации могут не сойтись — просто пропускаем
        continue

print(f"Лучшая модель ARIMA{best_order} с AIC = {best_aic:.2f}")

# если вдруг best_model не нашёлся
if best_model is None:
    raise RuntimeError("Не удалось подобрать модель ARIMA — попробуй расширить/изменить сетку p,d,q.")

# ----- 3. Прогноз на 12 периодов вперёд -----

steps = 12
forecast_res = best_model.get_forecast(steps=steps)
forecast = forecast_res.predicted_mean
conf_int = forecast_res.conf_int()

# собираем прогноз в отдельный датафрейм
forecast_df = pd.DataFrame({
    'date': forecast.index,
    'forecast': forecast.values,
    'lower_ci': conf_int.iloc[:, 0].values,
    'upper_ci': conf_int.iloc[:, 1].values
})

print(forecast_df)

# при необходимости — выгрузка в Excel
# forecast_df.to_excel("forecast_arima_12.xlsx", index=False)

# ----- 4. График: история + прогноз + доверительный интервал -----

plt.figure(figsize=(12, 6))

# исторические значения
plt.plot(ts.index, ts.values, label="Исторические данные")

# прогноз
plt.plot(forecast.index, forecast.values, linestyle='--', label="Прогноз (ARIMA)")

# доверительный интервал
plt.fill_between(
    forecast.index,
    conf_int.iloc[:, 0],
    conf_int.iloc[:, 1],
    alpha=0.2,
    label="Доверительный интервал"
)

plt.title(f"Прогноз ARIMA{best_order} на {steps} периодов вперёд")
plt.xlabel("Дата")
plt.ylabel(value_col)
plt.legend()
plt.grid(True)
plt.tight_layout()
plt.show()
