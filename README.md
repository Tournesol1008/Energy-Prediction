# Background
The number of prosumers is rapidly increasing, and solving the problems of energy imbalance and their rising costs is vital. If left unaddressed, this could lead to increased operational costs, potential grid instability, and inefficient use of energy resources. If this problem were effectively solved, it would significantly reduce the imbalance costs, improve the reliability of the grid, and make the integration of prosumers into the energy system more efficient and sustainable. Moreover, it could potentially incentivize more consumers to become prosumers, knowing that their energy behavior can be adequately managed, thus promoting renewable energy production and use.
# Goal Overview
Our goal is to creat an energy prediction model of prosumers to predict the amount of electricity produced and consumed by Estonian energy customers who have installed solar panels. We'll have access to weather data, the relevant energy prices, and records of the installed photovoltaic capacity.
# Dataset Description
## train.csv

This is our main backbone dataset, next we generate features and join them to this dataset for training.


- `county` - An ID code for the county.
- `is_business` - Boolean for whether or not the prosumer is a business.
- `product_type` - ID code with the following mapping of codes to contract types: `{0: "Combined", 1: "Fixed", 2: "General service", 3: "Spot"}`.
- `target` - The consumption or production amount for the relevant segment for the hour. The segments are defined by the `county`, `is_business`, and `product_type`.
- `is_consumption` - Boolean for whether or not this row's target is consumption or production.
- `datetime` - The Estonian time in EET (UTC+2) / EEST (UTC+3).
- `data_block_id` - All rows sharing the same data_block_id will be available at the same forecast time. This is a function of what information is available when forecasts are actually made, at 11 AM each morning. For example, if the forecast weather `data_block_id` for predictins made on October 31st is 100 then the historic weather `data_block_id` for October 31st will be 101 as the historic weather data is only actually available the next day.
- `row_id` - A unique identifier for the row.
- `prediction_unit_id` - A unique identifier for the `county`, `is_business`, and `product_type` combination. New prediction units can appear or disappear in the test set.
## weather_station_to_county_mapping.csv

`forecast_weather` and `historical_weather` files are generated from several weather station (with latitude and longitude). 
![](/figures/Weather_Station.png)
From the following map plot we see that for some counties we have more than one weather station, also we have data from weather stations that far from counties (in grey dot).

## historical_weather.csv

- `datetime`
- `temperature`
- `dewpoint`
- `rain` - Different from the forecast conventions. The rain from large scale weather systems of the preceding hour in millimeters.
- `snowfall` - Different from the forecast conventions. Snowfall over the preceding hour in centimeters.
- `surface_pressure` - The air pressure at surface in hectopascals.
- `cloudcover_[low/mid/high/total]` - Different from the forecast conventions. Cloud cover at 0-3 km, 3-8, 8+, and total.
- `windspeed_10m` - Different from the forecast conventions. The wind speed at 10 meters above ground in meters per second.
- `winddirection_10m` - Different from the forecast conventions. The wind direction at 10 meters above ground in degrees.
- `shortwave_radiation` - Different from the forecast conventions. The global horizontal irradiation in watt-hours per square meter.
- `direct_solar_radiation`
- `diffuse_radiation` - Different from the forecast conventions. The diffuse solar irradiation in watt-hours per square meter.
- `[latitude/longitude]` - The coordinates of the weather station.
- `data_block_id`

## forecast_weather.csv 
Weather forecasts that would have been available at prediction time. Sourced from the European Centre for Medium-Range Weather Forecasts.

- `[latitude/longitude]` - The coordinates of the weather forecast.
- `origin_datetime` - The timestamp of when the forecast was generated.
- `hours_ahead` - The number of hours between the forecast generation and the forecast weather. Each forecast covers 48 hours in total.
- `temperature` - The air temperature at 2 meters above ground in degrees Celsius.
- `dewpoint` - The dew point temperature at 2 meters above ground in degrees Celsius.
- `cloudcover_[low/mid/high/total]` - The percentage of the sky covered by clouds in the following altitude bands: 0-2 km, 2-6, 6+, and total.
- `10_metre_[u/v]_wind_component` - The [eastward/northward] component of wind speed measured 10 meters above surface in meters per second.
- `data_block_id`
- `forecast_datetime` - The timestamp of the predicted weather. Generated from origin_datetime plus hours_ahead.
- `direct_solar_radiation` - The direct solar radiation reaching the surface on a plane perpendicular to the direction of the Sun accumulated during the preceding hour, in watt-hours per square meter.
- `surface_solar_radiation_downwards` - The solar radiation, both direct and diffuse, that reaches a horizontal plane at the surface of the Earth, in watt-hours per square meter.
- `snowfall` - Snowfall over the previous hour in units of meters of water equivalent.
- `total_precipitation` - The accumulated liquid, comprising rain and snow that falls on Earth's surface over the preceding hour, in units of meters.

## client.csv

- `product_type`
- `county` - An ID code for the `county`. See `county_id_to_name_map.json` for the mapping of ID codes to county names.
- `eic_count` - The aggregated number of consumption points (EICs - European Identifier Code).
- `installed_capacity` - Installed photovoltaic solar panel capacity in kilowatts.
- `is_business` - Boolean for whether or not the prosumer is a business.
- `date`
- `data_block_id`

## electricity_prices.csv

- `origin_date`
- `forecast_date`
- `euros_per_mwh` - The price of electricity on the day ahead markets in euros per megawatt hour.
- `data_block_id`

## gas_prices.csv

- `origin_date` - The date when the day-ahead prices became available.
- `forecast_date` - The date when the forecast prices should be relevant.
- `[lowest/highest]_price_per_mwh` - The lowest/highest price of natural gas that on the day ahead market that trading day, in Euros per megawatt hour equivalent.
- `data_block_id`





# Evaluation
Prediction accuracy are evaluated on the Mean Absolute Error (MAE) between the predicted return and the observed target. The formula is given by:
$${\rm MAE} = \frac{1}{n} \sum_{i=1}^n |y_i - x_i|, $$

Where: 
- $n$ is the total number of data points, 
- $y_i$ is the predicted value for data point $i$, 
- $x_i$ is the observed value for data point $i$.

---

# <span style="color:red;">Baseline</span>
> ## What do we need to predict?
For each county we need to make predicitons of consumption or production of electricity.
For each county we need to make prediction for several subgroups
`is_business(True/False)`, `is_consumption(True/False)`, `product_type(0, 1, 2, 3)`.

Let's call a combination of `county`, `is_business`, `is_consumption`, `product_type` as a segment.

We need to forecast each day at `11:00:00` (through the special API we get new available data each day, send forecasts).
Imagine that we at `2023-05-27 11:00:00`.

> ## Timeline of availability of the data
Let’s say we are on day D at 11am. We want to predict next day D+1 net consumption from 00 to 23 for every hours.
| Category                         | Weather forecast                                    | Historical weather                    | Historical consumption and production / Client data   | Electricity prices                         | Gas prices                |
|----------------------------------|-----------------------------------------------------|---------------------------------------|-------------------------------------------------------|--------------------------------------------|---------------------------|
| Last available data              | Forecast for every hours of D and D+1 (published on D) | Every hours until day D, 10 am         | Every hours of Day D-1                                | Every hours of day D (published on D-1)    | Data for day D (published on D-1) |

Prices are published everyday at 2 pm (so after 11 am), that is we do not have D+1 prices.

**Current time** 
`2023-05-27 11:00:00`
        
We need to predict for each hour and each segment next day
`2023-05-28 00:00:00` 
`2023-05-28 01:00:00` 
`...` 
`2023-05-28 23:00:00`
        
At our current time we have acces to new additional data

**revealed_targets** (factual data for the previous day forecast) 
`2023-05-26 00:00:00` 
`2023-05-26 01:00:00` 
`...` 
`2023-05-26 23:00:00`
  

**client** (`installed_capacity`, `eic_count`) 
`2023-05-26`


**historical_weather** 
`2023-05-26 11:00:00` 
`2023-05-26 12:00:00` 
`...` 
`2023-05-27 10:00:00` 

        
**forecast_weather** (made at `2023-05-27 00:00:00`)
`2023-05-27 01:00:00` 
`2023-05-27 02:00:00` 
`...` 
`2023-05-29 23:00:00`

        
**electricity_prices**
`2023-05-27 00:00:00` 
`2023-05-27 01:00:00` 
`...` 
`2023-05-27 23:00:00`

        
**gas_prices** 
`2023-05-27`
        
    
Of course we can use all data which was available before `2023-05-27 11:00:00` also.

        
Understanding the logic behind forecast process helps to create a correct dataset for training our models 
(without unavailale data at that point).