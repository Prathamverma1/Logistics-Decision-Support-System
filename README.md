# Logistics Decision-Support System: Demand Forecasting, Truck Allocation & Route Optimization

This project builds three connected components of a logistics decision-support pipeline: a shipment/weight demand forecasting model, a capacity-constrained truck allocation engine, and a nearest-neighbor route optimizer, applied to a simulated Indian last-mile delivery dataset.

---

## Project Overview

The dataset represents ~25,000 delivery records across multiple courier partners (Delhivery, Xpressbees, Shadowfax, etc.), covering package type, vehicle type, delivery mode, region, weather, distance, weight, delay status, rating, and cost. On top of this, the project simulates order dates and delivery timelines to enable time-series demand forecasting, then layers on operational logic for truck loading and route sequencing.

The goal was to go beyond descriptive analysis and build the operational pieces a logistics team would actually use: *how many shipments are coming, which truck should carry which order, and what order should a truck visit its stops in.*

---

## Components

### 1. Demand Forecasting (XGBoost)
Two separate regressors were trained:
- **Shipment count forecast** — predicts daily number of shipments
- **Package weight forecast** — predicts total daily package weight

Features engineered per day: calendar features (month, week of year, quarter, weekend/month-start/month-end flags), lag features (1/7/14/28 days), rolling means (7/14/28 days), rolling std, and an exponentially weighted moving average. Both models use `XGBRegressor` (200 estimators, depth 5, learning rate 0.05) with an 80/20 train-test split, evaluated on R², MAE, and MSE.

A recursive forecasting function extends predictions day-by-day, feeding each day's prediction back in as history to compute the next day's lag/rolling features.

### 2. Truck Allocation Engine
A capacity-constrained bin-packing routine: given a fleet of trucks (each with a capacity and any pre-loaded orders) and a batch of new orders, orders are sorted heaviest-first and greedily assigned to the first truck with sufficient remaining capacity. Orders that don't fit anywhere are returned as unallocated so the team knows what needs another truck or another day.

### 3. Route Optimization
A nearest-neighbor heuristic builds a delivery sequence per truck starting and ending at a central warehouse (Kanpur), visiting existing (pre-loaded) destinations first, then greedily choosing the nearest remaining destination by haversine distance until all assigned stops are covered. Coordinates are mapped across 25 major Indian cities grouped into five regions (north, south, east, west, central).

### 4. Scenario Planning
A demand-scenario function estimates fleet needs under a hypothetical demand increase (e.g., "what if demand rises 12%?"), combining forecasted and actual month-to-date weight to calculate whether additional trucks are required given a fleet size and average truck capacity.

---

## Tools Used

- **Python**: pandas, NumPy for data prep and feature engineering
- **scikit-learn**: train/test split, evaluation metrics
- **XGBoost**: demand forecasting models
- **Haversine distance**: route sequencing
- **Google Colab**: development environment

---

## Known Limitations (worth stating plainly)

- **Dates are synthetic.** `order_date` is randomly generated over a fixed range rather than pulled from real historical timestamps, so the forecasting models are learning patterns in simulated seasonality, not actual demand history.
- **Allocation is a greedy heuristic, not an optimizer.** It doesn't guarantee minimal truck count or optimal packing — a proper bin-packing solver (e.g., OR-Tools) would do better.
- **Routing is nearest-neighbor, not a real routing engine.** No road network, traffic, or time-window constraints — it's straight-line (haversine) distance between 25 fixed city centroids.
- **No dashboard currently exists.** The pipeline outputs are printed/DataFrame results and per-truck route lists; there's no interactive or executive-facing visualization layer yet.
- **Scripts are interactive (Colab `input()` prompts)**, not yet packaged as a reusable pipeline or API.

---

## Possible Next Steps

- Replace synthetic dates with a realistic order-arrival simulation (or real data if available) before trusting the forecast for planning
- Swap the greedy allocator for a proper bin-packing/assignment optimizer
- Integrate a real distance/routing API for road-network-aware sequencing
- Build a lightweight dashboard (Streamlit or Tableau) over the forecast, allocation, and route outputs
- Refactor `input()`-based scripts into functions that take structured arguments, for reuse outside Colab
