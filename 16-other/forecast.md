# Amazon Forecast

> **Status (2026):** Amazon Forecast is **no longer available to new customers** (since July 25, 2024). Existing customers can continue to use it. AWS recommends **Amazon SageMaker Canvas** (time-series forecasting, no-code) or SageMaker AutoML/AutoGluon for new workloads.

- Provides forecasting for time series data. For example can be used for predicting retail demand, supply chain, staffing levels, energy requirements, server capacity and web traffic
- For any type of time-series data where we have a large amount of historical data, Forecast can be used to provide the ability to forecast future trends and events
- In order to accomplish this forecasting we have to import historical data and other related data. Related data includes extra contextual information (example promotions running when an item was sold)
- The output of Forecast will be a forecast and forecast explainability. The explainability allows to understand the reasons behind the forecasted data
- Workflow: import datasets (target time series + related time series + item metadata) from S3 -> train a **predictor** (AutoPredictor picks/ensembles algorithms: CNN-QR, DeepAR+, Prophet, NPTS, ARIMA, ETS) -> generate forecasts (probabilistic quantiles, e.g. P10/P50/P90) -> export to S3 or query via API
- What-if analysis: simulate scenarios (e.g. price changes) on the related time series
- Accuracy metrics: WAPE, RMSE, MAPE, MASE, weighted quantile loss
- Exam: "forecast demand from historical time-series data without ML expertise" -> today SageMaker Canvas; Forecast only if already in use
