---
title: "How to Scrape and Prepare PCSO Lottery Data for Analysis in Excel"
category: ["Data Science", "Code"]
date: 2021-08-05T00:00:00Z
lastmod: 2022-05-09T00:00:00Z
description: "In this post we will be scraping the historical lottery data for various games at the Philippine Charity Sweepstakes Office website so we can analyze it for later."
math: false
showtoc: true
slug: "scraping-lottery-data"
summary: "Scraping the Philippine Charity Sweepstakes Office website for historical lottery data of various games for later analysis."
tags: ["code", "data science", "web scraping", "microsoft excel", "python", "selenium"]
cover:
    alt: "Lottery tickets (decorative)"
    caption: 
    image: "https://res.cloudinary.com/jskherman/image/upload/v1651554590/Paper/lottery-waldemar-brandt-unsplash_csypd6.jpg"
    relative: false
---

{{< load-photoswipe >}}

In this post we will be scraping the historical lottery data for various games at the Philippine Charity Sweepstakes Office website so we can analyze it for later.

# Scraping the Data

------------------------------------------------------------------------

Let us first install the modules we're going to need for scraping the website (skip this if you already have them).

``` python
python -m pip install selenium beautifulsoup4 pandas
```

And then import them for our project:

``` python
# Selenium for simulating clicks in a browser
from selenium import webdriver
from selenium.webdriver.support.ui import Select
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as ec

# BeautifulSoup for scraping
from bs4 import BeautifulSoup

# pandas for processing the data
import pandas as pd

# datetime for getting today's date and formatting
from datetime import datetime
from datetime import date
from datetime import datetime, timedelta
```

## Simulate clicks in the Browser with `selenium`

For simulating clicks in a web broswer, we are going to use `selenium`. We're also going to need to download a web driver such as Microsoft Edge's [webdriver](https://docs.microsoft.com/en-us/microsoft-edge/webdriver-chromium/) (of course, you need the corresponsing browser [Microsoft Edge](https://www.microsoft.com/en-us/edge) to be installed). Since I'm on Windows, MS Edge comes right out of the box so I will use it, but feel free to use your own preferred browser that has a driver such as [Chrome](https://chromedriver.chromium.org/home) or [Firefox](https://github.com/mozilla/geckodriver/releases).

Now set the path to where the webdriver executuble is as well as the url to the lottery data is located.

``` python
# Set the path to where the webdriver executuble is
path = ("C:\\Users\\<Username> Herman\\AppData\\Local\\Programs\\Python" +
         "\\Python39\\Scripts\\msedgedriver.exe")

# Designate the url to be scraped to a variable
url = "https://www.pcso.gov.ph/SearchLottoResult.aspx"
```

Now initialize a Selenium session by directing it to the webdriver executable.

``` python
# Initialize the Edge webdriver
driver = webdriver.Edge(executable_path=path) 

# Grab the web page
driver.get(url)                              
```

One problem with the page though is that if you inspect the page's source there is class called `pre-con` in a div. If you would try to just have the driver proceed without waiting for a few seconds some of the buttons are unclickable and blocked by this div container, so we have to tell the WebDriver to wait for a set amount of time. I discovered this after a while of troubleshooting why the selenium cannot give any input to the web form.

``` python
# Designate a variable for waiting for the page to load
wait = WebDriverWait(driver, 5)

# wait for the div class "pre-con" to be invisible to ensure clicks work
wait.until(ec.invisibility_of_element_located((By.CLASS_NAME, "pre-con")))
```

    [OUT]: <selenium.webdriver.remote.webelement.WebElement (session="fb056d28fee8b152833d3ed6d8827c99", element="a7ca8ad3-7f46-47c8-8b8b-1b80414f1e51")>

Now that the wait is over let us now proceed to entering our parameters in the ASP.NET web form (the form with filters we see if we visit the web page) for the data we need. We are going to do that by using the `.find_element_by_id` method here for the options because we know their `id`s by inspecting the page's source at the place where the dropdown menus are.

Tip: To inspect the dropdown menu, right-click on it and navigate to "Developer Tools" and select "Inspect" (or press `F12`). We then get the value inside of the `id` parameter.

{{< figure src="https://res.cloudinary.com/jskherman/image/upload/v1651553052/Paper/search-lotto_lble83.png" alt="The Search Lotto Form" >}}

We want the end date to be today to get the latest data from all the games and the start date to be the earliest possible option which is `January 1, 2011` in the dropdown menu. As for the lotto game we want all games. We will just split the data up later into smaller dataframes using pandas for each lotto game, so that we only need to scrape the website every time we want to update our data. But first get today's date which will be used later.

``` python
# Get today's date with the datetime import
today = date.today()

# Store the current year, month, and day to variables
td_year = today.strftime("%Y")
td_month = today.strftime("%B")
td_day = today.strftime("%d").lstrip("0").replace(" 0", " ")

startyr = int(td_year) - 10
startyr = str(startyr)
print("Today is " + td_month + " " + td_day + ", " + td_year + ".\n")
```

    [OUT]: Today is May 9, 2022.

Now let's have Selenium and the webdriver find the elements of the form and select the parameters we want in the form options.

``` python
# Select Start Date as January 1, 2011
start_month = Select(driver.find_element_by_id(
    "cphContainer_cpContent_ddlStartMonth"))
start_month.select_by_value("January")

start_day = Select(driver.find_element_by_id(
    "cphContainer_cpContent_ddlStartDate"))
start_day.select_by_value("1")

start_year = Select(driver.find_element_by_id(
    "cphContainer_cpContent_ddlStartYear"))
start_year.select_by_value(startyr)

# Select End Date as Today
end_month = Select(driver.find_element_by_id(
    "cphContainer_cpContent_ddlEndMonth"))
end_month.select_by_value(td_month)

end_day = Select(driver.find_element_by_id("cphContainer_cpContent_ddlEndDay"))
end_day.select_by_value(td_day)

end_year = Select(driver.find_element_by_id(
    "cphContainer_cpContent_ddlEndYear"))
end_year.select_by_value(td_year)

# Lotto Game
game = Select(driver.find_element_by_id(
    "cphContainer_cpContent_ddlSelectGame"))

# If you inspect the page, the value of the option for "All Games" is 0
game.select_by_value('0')

# Submit the parameters by clicking the search button
search_button = driver.find_element_by_id("cphContainer_cpContent_btnSearch")
search_button.click()
```

## Scraping the data using `BeautifulSoup`

Now it's time to scrape the data from the current page's session with `BeautifulSoup` to get the data we need.

Firstly, feed the page's source code into Beautiful Soup and then have it find our results by `id`. Inspect the source again) and get all the table's rows by their attributes such as `class`.

``` python
# Feed the page's source code into Beautiful Soup
doc = BeautifulSoup(driver.page_source, "html.parser")

# Find the table of the results by id (
rows = doc.find('table', id='cphContainer_cpContent_GridView1').find_all(
    'tr', attrs={'class': "alt"})
```

Now time to put the data in a python list/dictionary.

``` python
# Initialize a list to hold our data
entries = []

# Now loop through the rows and put the data into the list to make a table
for row in rows:
    cells = row.select("td")
    entry = {
        "Game": cells[0].text,
        "Combination": cells[1].text,
        "Date": cells[2].text,
        "Prize": cells[3].text,
        "Winners": cells[4].text,
    }
    entries.append(entry)
```

# Processing the Data

------------------------------------------------------------------------

## Cleaning Up the Data with `pandas`

Now that we have the data in a list, it is now time to put it in a `pandas` dataframe and clean it up. There are duplicates in the data if you examine it closely so we have to remove those. We also need to get the data into the proper data types to make it easier for us to process down the line (i.e. sanitization).

``` python
# Turn the list into a dataframe
df = pd.DataFrame(entries)

# Remove duplicate rows
df.drop_duplicates(inplace=True, keep=False)

# Remove rows that have no combination associated
df = df[df["Combination"] != "-                "]
```

The part `df = df[df["Combination"] != "- "]` above is to look for and remove entries that do not have a combination. I also found this after hours of figuring out why I cannot do certain operations on the data like converting them into the proper data types. Speaking of data types, let's go convert the data now.

``` python
# Convert the dates to datetime type
df["Date"] = df["Date"].astype('datetime64[ns]')

# Remove the commas in the prize amounts
df["Prize"] = df["Prize"].replace(',','', regex=True)
    
# Convert data types of Prize and Winners to float and integers
df["Prize"] = df["Prize"].astype(float)     # float because there are still centavos
df["Winners"] = df["Winners"].astype(int)
```

Now let's look at our data so far:

``` python
df
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Game</th>
      <th>Combination</th>
      <th>Date</th>
      <th>Prize</th>
      <th>Winners</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Superlotto 6/49</td>
      <td>18-24-04-26-47-36</td>
      <td>2022-05-08</td>
      <td>67522822.8</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Suertres Lotto 4PM</td>
      <td>1-3-9</td>
      <td>2022-05-08</td>
      <td>4500.0</td>
      <td>279</td>
    </tr>
    <tr>
      <th>2</th>
      <td>EZ2 Lotto 11AM</td>
      <td>08-04</td>
      <td>2022-05-08</td>
      <td>4000.0</td>
      <td>251</td>
    </tr>
    <tr>
      <th>3</th>
      <td>EZ2 Lotto 9PM</td>
      <td>07-04</td>
      <td>2022-05-08</td>
      <td>4000.0</td>
      <td>784</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Lotto 6/42</td>
      <td>14-24-08-16-22-37</td>
      <td>2022-05-07</td>
      <td>6051682.0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>15880</th>
      <td>4D Vismin</td>
      <td>0-6-3-3</td>
      <td>2012-01-02</td>
      <td>40672.0</td>
      <td>7</td>
    </tr>
    <tr>
      <th>15881</th>
      <td>Suertres Lotto 4PM</td>
      <td>3-9-3</td>
      <td>2012-01-02</td>
      <td>4500.0</td>
      <td>256</td>
    </tr>
    <tr>
      <th>15882</th>
      <td>EZ2 Lotto 9PM</td>
      <td>03-13</td>
      <td>2012-01-02</td>
      <td>4000.0</td>
      <td>540</td>
    </tr>
    <tr>
      <th>15883</th>
      <td>EZ2 Lotto 11AM</td>
      <td>31-24</td>
      <td>2012-01-02</td>
      <td>4000.0</td>
      <td>68</td>
    </tr>
    <tr>
      <th>15884</th>
      <td>Grand Lotto 6/55</td>
      <td>44-14-51-52-39-08</td>
      <td>2012-01-02</td>
      <td>71768080.8</td>
      <td>0</td>
    </tr>
  </tbody>
</table>
<p>15878 rows × 5 columns</p>
</div>

## Saving the data to an MS Excel workbook

So far it's looking good. Since we're now here it's time for us to split this huge dataframe of ours into smaller dataframes by the type of lotto game. While we're at it let's also fix the time for the Suertres Lotto and EZ2 Lotto games so that they are included in the data and not as a separate category.

After doing that, let's save that into an Excel workbook so that we do not have to scrape every time we want to analyze the data.

``` python
# Sort the dataframe by game
df.sort_values(by=["Game"], inplace=True)

# Get a list of the games
games = df["Game"].unique().tolist()

# Now we can create dataframes for each game
lotto_658 = df.loc[df["Game"]=="Ultra Lotto 6/58"].copy()
lotto_655 = df.loc[df["Game"]=="Grand Lotto 6/55"].copy()

# Sidenote: Super Lotto 6/49 and Mega Lotto 6/45 have different values from
# what was on the dropdown menu
lotto_649 = df.loc[df["Game"]=="Superlotto 6/49"].copy()
lotto_645 = df.loc[df["Game"]=="Megalotto 6/45"].copy()

# Anyways, continuing...
lotto_642 = df.loc[df["Game"]=="Lotto 6/42"].copy()
lotto_6d = df.loc[df["Game"]=="6Digit"].copy()
lotto_4d = df.loc[df["Game"]=="4Digit"].copy()

lotto_3da = df.loc[df["Game"]=="Suertres Lotto 11AM"].copy()
lotto_3db = df.loc[df["Game"]=="Suertres Lotto 4PM"].copy()
lotto_3dc = df.loc[df["Game"]=="Suertres Lotto 9PM"].copy()

lotto_2da = df.loc[df["Game"]=="EZ2 Lotto 11AM"].copy()
lotto_2db = df.loc[df["Game"]=="EZ2 Lotto 4PM"].copy()
lotto_2dc = df.loc[df["Game"]=="EZ2 Lotto 9PM"].copy()
```

Let's look at one of the dataframes:

``` python
lotto_2da.tail()
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Game</th>
      <th>Combination</th>
      <th>Date</th>
      <th>Prize</th>
      <th>Winners</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>11555</th>
      <td>EZ2 Lotto 11AM</td>
      <td>29-28</td>
      <td>2014-09-24</td>
      <td>4000.0</td>
      <td>54</td>
    </tr>
    <tr>
      <th>15533</th>
      <td>EZ2 Lotto 11AM</td>
      <td>27-16</td>
      <td>2012-03-12</td>
      <td>4000.0</td>
      <td>55</td>
    </tr>
    <tr>
      <th>15563</th>
      <td>EZ2 Lotto 11AM</td>
      <td>03-08</td>
      <td>2012-03-06</td>
      <td>4000.0</td>
      <td>206</td>
    </tr>
    <tr>
      <th>9533</th>
      <td>EZ2 Lotto 11AM</td>
      <td>24-30</td>
      <td>2016-01-06</td>
      <td>4000.0</td>
      <td>121</td>
    </tr>
    <tr>
      <th>2394</th>
      <td>EZ2 Lotto 11AM</td>
      <td>27-08</td>
      <td>2020-11-07</td>
      <td>4000.0</td>
      <td>127</td>
    </tr>
  </tbody>
</table>
</div>

For the Suertres Lotto and EZ2 Lotto games the games are split into 11:00 AM, 4:00 PM, and 9:00 PM games. Let's fix that by assigning them the proper datetime values in the Date column and combining them into bigger dataframes

``` python
# Add 11 hours to the datetime for Suertres Lotto 11AM game to match because of timezones
lotto_3da["Date"] = lotto_3da["Date"] + timedelta(hours=11)

# Add 16 hours to the datetime for Suertres Lotto 4PM game to match
lotto_3db["Date"] = lotto_3db["Date"] + timedelta(hours=16)

# Add 21 hours to the datetime for Suertres Lotto 9PM game to match
lotto_3dc["Date"] = lotto_3dc["Date"] + timedelta(hours=21)

# Rename all the game entries as just Suertres Lotto
lotto_3da["Game"] = "Suertres Lotto"
lotto_3db["Game"] = "Suertres Lotto"
lotto_3dc["Game"] = "Suertres Lotto"

# Combine the three Suertres Lotto dataframes into one
lotto_3d = lotto_3da
lotto_3d = lotto_3d.append(lotto_3db)
lotto_3d = lotto_3d.append(lotto_3dc)
```

``` python
# Do the same for EZ2 Lotto
lotto_2da["Date"] = lotto_2da["Date"] + timedelta(hours=11)
lotto_2db["Date"] = lotto_2db["Date"] + timedelta(hours=16)
lotto_2dc["Date"] = lotto_2dc["Date"] + timedelta(hours=21)

# Rename all the game entries as just EZ2 Lotto
lotto_2da["Game"] = "EZ2 Lotto"
lotto_2db["Game"] = "EZ2 Lotto"
lotto_2dc["Game"] = "EZ2 Lotto"

# Combine the three EZ2 Lotto dataframes into one
lotto_2d = lotto_2da
lotto_2d = lotto_2d.append(lotto_2db)
lotto_2d = lotto_2d.append(lotto_2dc)
```

Now let's look at one of them again to see if we were successful:

``` python
lotto_2da
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Game</th>
      <th>Combination</th>
      <th>Date</th>
      <th>Prize</th>
      <th>Winners</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2877</th>
      <td>EZ2 Lotto</td>
      <td>25-06</td>
      <td>2020-02-21 11:00:00</td>
      <td>4000.0</td>
      <td>54</td>
    </tr>
    <tr>
      <th>3858</th>
      <td>EZ2 Lotto</td>
      <td>05-06</td>
      <td>2019-07-18 11:00:00</td>
      <td>4000.0</td>
      <td>164</td>
    </tr>
    <tr>
      <th>7987</th>
      <td>EZ2 Lotto</td>
      <td>30-25</td>
      <td>2016-12-24 11:00:00</td>
      <td>4000.0</td>
      <td>223</td>
    </tr>
    <tr>
      <th>3876</th>
      <td>EZ2 Lotto</td>
      <td>29-29</td>
      <td>2019-07-14 11:00:00</td>
      <td>4000.0</td>
      <td>231</td>
    </tr>
    <tr>
      <th>14423</th>
      <td>EZ2 Lotto</td>
      <td>19-25</td>
      <td>2012-11-12 11:00:00</td>
      <td>4000.0</td>
      <td>135</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>11555</th>
      <td>EZ2 Lotto</td>
      <td>29-28</td>
      <td>2014-09-24 11:00:00</td>
      <td>4000.0</td>
      <td>54</td>
    </tr>
    <tr>
      <th>15533</th>
      <td>EZ2 Lotto</td>
      <td>27-16</td>
      <td>2012-03-12 11:00:00</td>
      <td>4000.0</td>
      <td>55</td>
    </tr>
    <tr>
      <th>15563</th>
      <td>EZ2 Lotto</td>
      <td>03-08</td>
      <td>2012-03-06 11:00:00</td>
      <td>4000.0</td>
      <td>206</td>
    </tr>
    <tr>
      <th>9533</th>
      <td>EZ2 Lotto</td>
      <td>24-30</td>
      <td>2016-01-06 11:00:00</td>
      <td>4000.0</td>
      <td>121</td>
    </tr>
    <tr>
      <th>2394</th>
      <td>EZ2 Lotto</td>
      <td>27-08</td>
      <td>2020-11-07 11:00:00</td>
      <td>4000.0</td>
      <td>127</td>
    </tr>
  </tbody>
</table>
<p>1815 rows × 5 columns</p>
</div>

Let's see also one of the combined dataframes:

``` python
lotto_2d
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Game</th>
      <th>Combination</th>
      <th>Date</th>
      <th>Prize</th>
      <th>Winners</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2877</th>
      <td>EZ2 Lotto</td>
      <td>25-06</td>
      <td>2020-02-21 11:00:00</td>
      <td>4000.0</td>
      <td>54</td>
    </tr>
    <tr>
      <th>3858</th>
      <td>EZ2 Lotto</td>
      <td>05-06</td>
      <td>2019-07-18 11:00:00</td>
      <td>4000.0</td>
      <td>164</td>
    </tr>
    <tr>
      <th>7987</th>
      <td>EZ2 Lotto</td>
      <td>30-25</td>
      <td>2016-12-24 11:00:00</td>
      <td>4000.0</td>
      <td>223</td>
    </tr>
    <tr>
      <th>3876</th>
      <td>EZ2 Lotto</td>
      <td>29-29</td>
      <td>2019-07-14 11:00:00</td>
      <td>4000.0</td>
      <td>231</td>
    </tr>
    <tr>
      <th>14423</th>
      <td>EZ2 Lotto</td>
      <td>19-25</td>
      <td>2012-11-12 11:00:00</td>
      <td>4000.0</td>
      <td>135</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>4686</th>
      <td>EZ2 Lotto</td>
      <td>16-27</td>
      <td>2019-01-11 21:00:00</td>
      <td>4000.0</td>
      <td>402</td>
    </tr>
    <tr>
      <th>4681</th>
      <td>EZ2 Lotto</td>
      <td>15-08</td>
      <td>2019-01-12 21:00:00</td>
      <td>4000.0</td>
      <td>249</td>
    </tr>
    <tr>
      <th>12361</th>
      <td>EZ2 Lotto</td>
      <td>16-29</td>
      <td>2014-03-16 21:00:00</td>
      <td>4000.0</td>
      <td>530</td>
    </tr>
    <tr>
      <th>4695</th>
      <td>EZ2 Lotto</td>
      <td>06-20</td>
      <td>2019-01-09 21:00:00</td>
      <td>4000.0</td>
      <td>335</td>
    </tr>
    <tr>
      <th>4290</th>
      <td>EZ2 Lotto</td>
      <td>01-20</td>
      <td>2019-04-09 21:00:00</td>
      <td>4000.0</td>
      <td>339</td>
    </tr>
  </tbody>
</table>
<p>5334 rows × 5 columns</p>
</div>

So far so good. The final stretch here is going to be to save our data to Microsoft Excel and we can achieve that easily with `pandas` with `pandas.DataFrame.to_excel`.

``` python
# Create Excel writer object
writer = pd.ExcelWriter("lotto.xlsx")

# Write dataframes to excel worksheets
df.to_excel(writer, "All Data")
lotto_658.to_excel(writer, "Ultra Lotto 6-58")
lotto_655.to_excel(writer, "Grand Lotto 6-55")
lotto_649.to_excel(writer, "Super Lotto 6-49")
lotto_645.to_excel(writer, "Mega Lotto 6-45")
lotto_642.to_excel(writer, "Lotto 6-42")

lotto_6d.to_excel(writer, "6 Digit")
lotto_4d.to_excel(writer, "4 Digit")
lotto_3d.to_excel(writer, "Suertres Lotto")
lotto_2d.to_excel(writer, "EZ2 Lotto")

# Save the Excel workbook
writer.save()
```

# Conclusion

With that we have saved an Excel file named `lotto.xlsx` where all of our scraped data have been put in. Now let's proceed to making the scripts for analyzing the Lotto data. For example, we might look at the frequency of combinations, how frequent some numbers appear compared to others, or the expectancy value is for each drawing date in part two.
