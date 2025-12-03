"""
Пример подбора гиперпараметров для ARIMA, SARIMA и Prophet
с помощью Scikit-Optimize (skopt) + комментарии.

Ожидаемые поля в test.xlsx:
- date    : дата наблюдения
- meaning : значение метрики
"""

import warnings

import numpy as np
import pandas as pd

from sklearn.metrics import mean_absolute_error

# Модели временных рядов
from statsmodels.tsa.arima.model import ARIMA
from statsmodels.tsa.statespace.sarimax import SARIMAX
from prophet import Prophet

# Scikit-Optimize (Bayesian Optimization на Gaussian Process)
from skopt import gp_minimize
from skopt.space import Integer, Real, Categorical

warnings.filterwarnings("ignore")


# ============================
# Загрузка и подготовка данных
# ============================

df = pd.read_excel("test.xlsx")

# Преобразуем дату (день первый)
df["date"] = pd.to_datetime(df["date"], dayfirst=True, errors="coerce")

# Приводим значения к числовому типу
df["meaning"] = pd.to_numeric(df["meaning"], errors="coerce")

# Чистим и сортируем
df = (
    df.dropna(subset=["date", "meaning"])
      .sort_values("date")
      .set_index("date")
)

if len(df) < 24:
    raise ValueError(f"Недостаточно наблюдений: {len(df)}. Нужен минимум 24.")

series = df["meaning"]

# Для Prophet нужна структура ds/y
data_prophet = series.reset_index()
data_prophet.columns = ["ds", "y"]


# ============================
# Train / Test сплит
# ============================

FORECAST_HORIZON = 12
test_steps = min(FORECAST_HORIZON, max(1, len(series) // 4))

train = series.iloc[:-test_steps]
test = series.iloc[-test_steps:]

train_p = data_prophet.iloc[:-test_steps].copy()
test_p = data_prophet.iloc[-test_steps:].copy()

print(f"Всего наблюдений: {len(series)}; train: {len(train)}, test: {len(test)}")


# ============================
# Метрика MAPE
# ============================

def mape(y_true, y_pred) -> float:
    """MAPE с защитой от деления на ноль (в %)."""
    y_true = np.array(y_true, dtype=float)
    y_pred = np.array(y_pred, dtype=float)
    eps = 1e-9
    denom = np.clip(np.abs(y_true), eps, None)
    return float(np.mean(np.abs((y_true - y_pred) / denom)) * 100)


# ============================================================
# 1. ARIMA + skopt (gp_minimize)
# ============================================================

"""
Scikit-Optimize (skopt) — базовая идея:
- мы описываем ПРОСТРАНСТВО поиска гиперпараметров (search space),
- пишем objective-функцию, которая для данного набора параметров
  обучает модель → считает ошибку (MAPE/MAE),
- gp_minimize строит Gaussian Process и "умно" выбирает следующие точки
  для проверки (в отличие от тупого перебора/рандома).
"""

# Пространство гиперпараметров ARIMA(p,d,q)
space_arima = [
    Integer(0, 6, name="p"),  # порядок AR
    Integer(0, 3, name="d"),  # порядок дифференцирования
    Integer(0, 6, name="q"),  # порядок MA
]


def objective_arima(params):
    """
    Целевая функция для ARIMA.
    params — список [p, d, q] с текущими значениями гиперпараметров,
    которые подбирает skopt.
    Возвращаем значение функции потерь (MAPE), которую нужно минимизировать.
    """
    p, d, q = params

    try:
        model = ARIMA(
            train,
            order=(p, d, q),
            enforce_stationarity=False,
            enforce_invertibility=False,
        )
        res = model.fit()
        forecast = res.get_forecast(steps=len(test)).predicted_mean
        return mape(test, forecast)
    except Exception:
        # Если модель "падает" или не сходится → возвращаем большой штраф
        return 1e9


print("\n=== ARIMA + Scikit-Optimize (gp_minimize) ===")

res_arima = gp_minimize(
    func=objective_arima,   # что оптимизируем
    dimensions=space_arima, # пространство поиска
    n_calls=30,             # сколько итераций (запусков objective)
    n_initial_points=5,     # сколько точек взять случайно до старта GP
    random_state=42         # для воспроизводимости
)

best_p, best_d, best_q = res_arima.x
print(f"Лучшие параметры ARIMA: p={best_p}, d={best_d}, q={best_q}")
print(f"Лучший MAPE (ARIMA): {res_arima.fun:.2f}%")

# Обучаем финальную ARIMA на полном train (можно и на всём ряду)
arima_final = ARIMA(
    train,
    order=(best_p, best_d, best_q),
    enforce_stationarity=False,
    enforce_invertibility=False,
).fit()

arima_fc_test = arima_final.get_forecast(steps=len(test)).predicted_mean
arima_mae = mean_absolute_error(test, arima_fc_test)
arima_mape = mape(test, arima_fc_test)

print(f"MAE  (ARIMA, test): {arima_mae:.4f}")
print(f"MAPE (ARIMA, test): {arima_mape:.2f}%")


# ============================================================
# 2. SARIMA + skopt (gp_minimize)
# ============================================================

"""
SARIMA(p,d,q)(P,D,Q,s):
- (p,d,q)    — обычные ARIMA-параметры
- (P,D,Q)    — сезонные параметры
- s          — длина сезона (например, 12 для месячных данных с годовой сезонностью)
"""

SEASONAL_PERIOD = 12  # предполагаем месячный ряд

space_sarima = [
    Integer(0, 3, name="p"),
    Integer(0, 2, name="d"),
    Integer(0, 3, name="q"),
    Integer(0, 2, name="P"),
    Integer(0, 1, name="D"),
    Integer(0, 2, name="Q"),
]


def objective_sarima(params):
    """
    Целевая функция для SARIMA.
    params = [p, d, q, P, D, Q]
    """
    p, d, q, P, D, Q = params

    try:
        model = SARIMAX(
            train,
            order=(p, d, q),
            seasonal_order=(P, D, Q, SEASONAL_PERIOD),
            enforce_stationarity=False,
            enforce_invertibility=False,
        )
        res = model.fit(disp=False)
        forecast = res.get_forecast(steps=len(test)).predicted_mean
        return mape(test, forecast)
    except Exception:
        return 1e9


print("\n=== SARIMA + Scikit-Optimize (gp_minimize) ===")

res_sarima = gp_minimize(
    func=objective_sarima,
    dimensions=space_sarima,
    n_calls=30,
    n_initial_points=6,
    random_state=42,
)

best_p, best_d, best_q, best_P, best_D, best_Q = res_sarima.x
print(
    f"Лучшие параметры SARIMA: "
    f"(p,d,q)=({best_p},{best_d},{best_q}), "
    f"(P,D,Q,s)=({best_P},{best_D},{best_Q},{SEASONAL_PERIOD})"
)
print(f"Лучший MAPE (SARIMA): {res_sarima.fun:.2f}%")

sarima_final = SARIMAX(
    train,
    order=(best_p, best_d, best_q),
    seasonal_order=(best_P, best_D, best_Q, SEASONAL_PERIOD),
    enforce_stationarity=False,
    enforce_invertibility=False,
).fit(disp=False)

sarima_fc_test = sarima_final.get_forecast(steps=len(test)).predicted_mean
sarima_mae = mean_absolute_error(test, sarima_fc_test)
sarima_mape = mape(test, sarima_fc_test)

print(f"MAE  (SARIMA, test): {sarima_mae:.4f}")
print(f"MAPE (SARIMA, test): {sarima_mape:.2f}%")


# ============================================================
# 3. Prophet + skopt (gp_minimize)
# ============================================================

"""
Для Prophet будем подбирать:
- seasonality_mode: 'additive' / 'multiplicative'
- changepoint_prior_scale: насколько "гибко" модель ловит изменения тренда
- seasonality_prior_scale: насколько сильно разрешаем сезонности
- yearly_seasonality: число гармоник Фурье для годовой сезонности
"""

space_prophet = [
    Categorical(["additive", "multiplicative"], name="seasonality_mode"),
    Real(0.01, 1.0, prior="log-uniform", name="changepoint_prior_scale"),
    Real(0.1, 10.0, prior="log-uniform", name="seasonality_prior_scale"),
    Integer(5, 15, name="yearly_seasonality"),
]


def objective_prophet(params):
    """
    Целевая функция для Prophet.
    params = [seasonality_mode, changepoint_prior_scale, seasonality_prior_scale, yearly_seasonality]
    """
    seasonality_mode, cps, sps, y_seas = params

    try:
        m = Prophet(
            growth="linear",
            weekly_seasonality=False,  # для месячных рядов недели часто не нужны
            daily_seasonality=False,
            seasonality_mode=seasonality_mode,
            changepoint_prior_scale=cps,
            seasonality_prior_scale=sps,
            yearly_seasonality=y_seas,
        )
        m.fit(train_p)

        future = m.make_future_dataframe(periods=len(test_p), freq="MS")
        fcst = m.predict(future).tail(len(test_p))

        return mape(test_p["y"], fcst["yhat"])
    except Exception:
        return 1e9


print("\n=== Prophet + Scikit-Optimize (gp_minimize) ===")

res_prophet = gp_minimize(
    func=objective_prophet,
    dimensions=space_prophet,
    n_calls=25,
    n_initial_points=5,
    random_state=42,
)

best_seasonality_mode, best_cps, best_sps, best_y_seas = res_prophet.x
print(
    "Лучшие параметры Prophet:",
    f"seasonality_mode={best_seasonality_mode},",
    f"changepoint_prior_scale={best_cps:.4f},",
    f"seasonality_prior_scale={best_sps:.4f},",
    f"yearly_seasonality={best_y_seas}",
)
print(f"Лучший MAPE (Prophet): {res_prophet.fun:.2f}%")

# Финальная модель Prophet
m_best = Prophet(
    growth="linear",
    weekly_seasonality=False,
    daily_seasonality=False,
    seasonality_mode=best_seasonality_mode,
    changepoint_prior_scale=best_cps,
    seasonality_prior_scale=best_sps,
    yearly_seasonality=best_y_seas,
)
m_best.fit(data_prophet)

future_test = m_best.make_future_dataframe(periods=len(test_p), freq="MS")
fcst_test = m_best.predict(future_test).tail(len(test_p))

prophet_mae = mean_absolute_error(test_p["y"], fcst_test["yhat"])
prophet_mape = mape(test_p["y"], fcst_test["yhat"])

print(f"MAE  (Prophet, test): {prophet_mae:.4f}")
print(f"MAPE (Prophet, test): {prophet_mape:.2f}%")


# ============================================================
# Итоговая таблица метрик
# ============================================================

metrics = pd.DataFrame(
    {
        "MAE": [arima_mae, sarima_mae, prophet_mae],
        "MAPE": [arima_mape, sarima_mape, prophet_mape],
    },
    index=["ARIMA + skopt", "SARIMA + skopt", "Prophet + skopt"],
)

print("\n=== Итоговая таблица метрик (test) ===")
print(metrics.round(4))
    
