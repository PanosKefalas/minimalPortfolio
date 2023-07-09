---
title: "Competitor price monitoring"
date: 09-07-2023
layout: single

toc: true
header: 
  image: /assets/images/priceScraper/cover.jpg
  teaser: /assets/images/priceScraper/cover.jpg
categories:
  - project
tags:
  - Webscraping
  - Python
  - BeautifoulSoup
  - Google Sheets API
  - Looker Studio
---

In this project, I developed a script that scrapes prices from an e-shop, stores the data in Google Sheets using the Google Sheets API, and generates reports on LookerStudio.

To accomplish this, I implemented several functions using Python and relevant libraries. Let's go through each step of the process:

## Step 1: Importing libraries

```python
import time
import gspread
from gspread_dataframe import get_as_dataframe, set_with_dataframe
import pandas as pd
import requests
from bs4 import BeautifulSoup
from datetime import date
```
- [time](https://docs.python.org/3/library/time.html): The time library provides various functions for working with time, such as getting the current time, adding delays, and measuring execution time. 
- [gspread](https://gspread.readthedocs.io/en/latest/): The gspread library is a Python client for Google Sheets API. It allows you to interact with Google Sheets, read and write data, create and modify spreadsheets, and perform various operations on the sheets.
- [pandas](https://pandas.pydata.org/docs/): The pandas library is a powerful data manipulation and analysis tool for Python. It provides data structures and functions to efficiently work with structured data, such as DataFrames, Series, and various data operations. gspread_dataframe relies on pandas to handle data manipulation.
- [requests](https://docs.python-requests.org/en/latest/): The requests library is a popular HTTP library for Python. It allows you to send HTTP requests, interact with web services, and retrieve data from web pages or APIs. 
- bs4 [(BeautifulSoup)](https://www.crummy.com/software/BeautifulSoup/bs4/doc/): The bs4 library, commonly referred to as BeautifulSoup, is a Python library for web scraping and parsing HTML or XML documents.
- [datetime](https://docs.python.org/3/library/datetime.html): The datetime library provides classes and functions to work with dates and times in Python. 


## Step 2: Scraping Functions

I utilized the requests library to send HTTP requests to the website and the BeautifulSoup library to parse the HTML content. By defining the **'getPrice'** function, I was able to extract the price information from product elements on the website. This function handled the necessary data cleaning and returned the price as a numeric value.

```python
def getPrice(product):
    prices = product.find(class_="price").text.strip()    
    try: price = prices.split(' ')[1]
    except IndexError:price = prices
    price = price.replace('€','').replace(',','.')
    
    return price
```

The **'getProductDetails'** function took a URL as input, retrieved the product details such as names and prices, and stored them in a pandas DataFrame.

```python
def getProductDetails(url):
    today = date.today()
    page = requests.get(url)
    soup = BeautifulSoup(page.content, 'html.parser')
    category = soup.find('h1', class_='heading-title').text
    products = soup.find_all('div', class_="product-details")
    names = []
    prices = []

    for product in products:        
        link = product.find('a').get('href')
        names.append(link.split('/')[-1])
        prices.append(getPrice(product))

    df = pd.DataFrame([prices], columns=names)
    df.insert(0, 'date', today)
        
    return df, category
```
## Step 3: Storing Data in Google Sheets

To store the scraped data in Google Sheets, I authenticated with the Google Sheets API and accessed the desired spreadsheet. Leveraging the **'gspread'** library, I developed the **'updateSheet'** function. This function allowed me to update a Google Sheets worksheet by appending the new data to the existing data and saving it. Additionally, I ensured that the updated DataFrame was locally saved as a CSV file for future reference.

```python
def updateSheet(sheet, category, newDf):
    worksheet = sheet.worksheet(category)
    oldDf = get_as_dataframe(worksheet).dropna(how='all').dropna(how='all', axis=1)
    updatedDf = pd.concat([oldDf, newDf])
    worksheet.clear()
    set_with_dataframe(worksheet, updatedDf)

    if save:
        today=date.today()
        updatedDf.to_csv(f'data/{category}/{today}.csv')
```

## Step 4: Bringing them all together

In the main script, I gathered in a list the URLs that needed to be scraped. I provided the necessary credentials file path and the key of the Google Sheets document to be updated. By iterating over each URL in the list, I called the **'getProductDetails'** function to retrieve the product details DataFrame and the category. Then, I passed these values, along with the Google Sheets worksheet, to the **'updateDf'** function, which handled the process of updating the worksheet with the new data.

I incorporated a 1-second delay between each URL request using the **'time.sleep'** function. This was done to ensure that the website's servers were not overloaded and to avoid potential blocking.

``````python
from scrapingFunctions import *
import time
import gspread


urlList = ['product1', 'product2', .... ]
credentials = 'credentials.json'
key = '********'


auth = gspread.service_account(filename=credentials)
sheet = auth.open_by_key(key)

for url in urlList:

    productsDf, category = getProductDetails(url)    
    updateDf(sheet, category, productsDf)
    
    time.sleep(1)
``````
**Eventually the google sheet shall look like this:**

![sheets](/assets/images/priceScraper/Screenshot from 2023-05-28 17-39-36.png)

**And the report on LookerStudio like this:**

![report](/assets/images/priceScraper/Screenshot from 2023-05-28 18-02-24.png)

Further customization is possible according to client requirements.  
Also an email notification could be sent when an update in the webpage has been spotted.

