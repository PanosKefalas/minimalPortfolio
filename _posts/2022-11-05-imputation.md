---
title: "Air Quality Data Analysis and Modelling - Part 1, Imputation"
date: 2022-11-05
layout: single

toc: true
header: 
  image: assets/images/airQuality/imputation/GettyImages-520120972-749c725865c24e11a194962a85c986ef.jpg
  teaser: assets/images/airQuality/imputation/GettyImages-520120972-749c725865c24e11a194962a85c986ef.jpg
categories:
  - project
tags:
  - Python
  - Data Preprocessing
  - Missing values
  - Imputation

---

*This is the first part of a series regarding Air Quality data analysis and modelling acquired through low cost sensors.*

## Introduction

Air pollution is considered the single most important environmental health risk factor in Europe. 

In this series, analysis is being performed and results outlined concerning the profile of air quality for the city of Thessaloniki, GR, according to data measured by Air Quality and Weather Monitoring Nodes located in the campus of Aristotle University of Thessaloniki.

It is not unusual that due to sensor malfunction or accidents data might be lost. This time series missing data can be reconstructed with Interpolation or Imputation methods. In this post we will try:


1. [Linear interpolation](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.interpolate.html)
2. [Cubic spline interpolation](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.interpolate.html)
3. [K-NN Imputation](https://scikit-learn.org/stable/modules/generated/sklearn.impute.KNNImputer.html)


## Importing the data.

Reading the csv file, obtained from the monitoring station, parsing dates, picking only the columns we need for a sample time.

```python
df_raw = pd.read_csv('station.csv', delimiter=';', decimal=',', parse_dates=[1], index_col=1).drop(columns=['Device'])
df = df_raw.loc['2022-09-05 00:00:00':,:'Temp ext (C)']
df.columns = (['Dew Point (C)', 'H2S (ppb)', 'NO2 (ppb)', 'Humidity (%)',  'O3 (ppb)', 'PM1 (ug/m3)', 'PM2.5 (ug/m3)', 'PM10 (ug/m3)', 'Temp (C)'])
df
```

The data sample that we are going to work with looks like this,

[![data table](/assets/images/airQuality/imputation/dataTable.png)](/assets/images/airQuality/imputation/dataTable.png)

and after checking for missing values, we notice that less than 5% of the Dew Point values are missing and almost none of the others:

[![missingVal](/assets/images/airQuality/imputation/missingValues.png)](/assets/images/airQuality/imputation/missingValues.png)



## Creating a test set

For this example the focus will be on NO2 that has no missing values at all, so we can test the imputation methods on clean data.



FIrst we are going to create a test set, by removing values on purpose, on 15 predefined intervals.

```python
target = 'NO2 (ppb)'
# working with the first 1000 values for simplicity.
df_test = df[:1000].copy()

nan_Index = []
intervals = 15
n_Missing_values = 12
# creating gaps of 12 missing values, at each index defined by nan_Index.
for interval in range(intervals+1):
    nan_Index.append(int(df_test.shape[0]/intervals*interval))

for i in nan_Index[1:]:
    df_test[target].iloc[i:i+n_Missing_values] = np.nan
```

Zooming in at two of the artificially crated data gaps:

![gapPlot](/assets/images/airQuality/imputation/gapPlot.png)

Now we are ready to start testing the imputation methods on the test set.

## K-NN Imputer

The K-NN imputer works by finding the K nearest neighbors of each sample with missing values and then using the values from those neighbors to impute the missing values. The algorithm calculates the distances between samples using a distance metric (usually Euclidean distance) and selects the K nearest neighbors based on those distances.

Once the K nearest neighbors are identified, the missing values are imputed by taking the average of the corresponding values in those neighbors.

We start by implementing the K-NN Imputer by [scikit-learn](https://jmlr.csail.mit.edu/papers/v12/pedregosa11a.html), with
- number of neighbours equal to 5
-  uniform weights 
- euclidean distance

Then calculating the evaluation metrics and saving them for later comparison:
- Mean Absolute Error
- Mean Square Error
- R2  

``````python
imputer = KNNImputer(n_neighbors=5)

df_knn = df_test.copy()
df_knn[:] = imputer.fit_transform(df_knn)

errors_list = [mean_absolute_error(df_knn[target], df[target][:1000]),
               mean_squared_error(df_knn[target], df[target][:1000]),
              r2_score(df_knn[target], df[target][:1000])]

errors_knn = pd.Series(errors_list,  index=['MAE','MSE', 'R2'])
``````

## Cubic Interpolation

In cubic interpolation, a cubic polynomial function is used to approximate the values between two adjacent data points. The polynomial is determined by considering the values of the two data points and their neighboring points. By using these four data points, the cubic polynomial is constructed to create a smooth curve that passes through all four points.

Implementing the built-in [Pandas](https://pandas.pydata.org/) cubic interpolation for a 3d order polynomial and calculating the same metrics:

```python
df_cubic = df_test.copy()
df_cubic.interpolate(method='cubic', order=3,inplace=True)

errors_list = [mean_absolute_error(df_cubic[target], df[target][:1000]),
               mean_squared_error(df_cubic[target], df[target][:1000]),
              r2_score(df_cubic[target], df[target][:1000])]

errors_cubic = pd.Series(errors_list,  index=['MAE','MSE', 'R2'])
```

## Linear Interpolation

In linear interpolation, a straight line is drawn between two adjacent data points, and the value at a specific position along that line is determined based on the relative distance between the known data points. The estimated value is calculated by taking into account the ratio of the distances of the target position from the two adjacent data points.

Similarly utilizing the linear interpolation:

``````python
df_linear = df_test.copy()
df_linear.interpolate(method='linear', inplace=True)

errors_list = [mean_absolute_error(df_linear[target], df[target][:1000]),
               mean_squared_error(df_linear[target], df[target][:1000]),
              r2_score(df_linear[target], df[target][:1000])]

errors_linear = pd.Series(errors_list,  index=['MAE','MSE', 'R2'])
``````
## Results

Plotting the same data gaps with each line representing a different filling method:
![filledGapPlot](/assets/images/airQuality/imputation/filledGapPlot.png)

And comparing the evaluation metrics we saved earlier for each method:

|        | MAE   | MSE   | R2    |
|--------|-------|-------|-------|
| K-NN   | 0.676 | 6.056 | 0.975 |
| Linear | 0.167 | 0.338 | 0.999 |
| Cubic  | 0.310 | 1.627 | 0.993 |

We notice that all three methods perform similarly, with Linear Interpolation resulting in the lowest errors.

A conclusion then would be that, for future missing NO2 values, Linear Interpolation would be a good choice for data reconstruction. 

> :warning: The same does not hold true for every period of the year, as we are processing time series data. Each method should be tested again for different periods.

Below are presented Plots for other features, besides NO2. We can see that each variable responds differently to K-NN Imputer, Linear and Cubic interpolation.

![h2sImputation](/assets/images/airQuality/imputation/H2S_Imputation.png)

