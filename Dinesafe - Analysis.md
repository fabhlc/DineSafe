
# Using DineSafe Data to Visualize Business Turnover
#### Fabienne Chan - January 3rd 2019 


---

# Index
### [Introduction](#intro)
### [Part I. Methodology](#methodology)
- ### [1.1 Data Origin](#data_origin)
- ### [1.2 Cleaning](#cleaning)
- ### [1.3 Feature Engineering](#feature_engineering)


### [Part II. Analysis](#analysis)
- ### [2.1 Closures](#closures)
- ### [2.2 Turnover](#turnover)
- ### [2.2.1 Turnover by Former Municipality](#turnover_by_fm)
- ### [2.2.2 Turnover by Neighbourhood](#turnover_by_n)
- ### [2.2.3 Turnover by Address](#turnover_by_a)

### [Conclusion](#conclusion)
#### [Footnotes](#footnote)

---

---
## Introduction <a class ="anchor" id = "intro"></a>

The purpose of this study is to visualize food-establishment turnover in the City of Toronto using DineSafe inspection data. Establishments that handle food are required to undergo **at least one inspection per year**. As such, it's fair to assume that the DineSafe data provides annual snapshots of all food-establishments in Toronto. Premised upon this, this analysis aims to visualize establishment turnover, which may act as a means to indicate the economic health of an area, particularly areas downtown. 

During the analysis, it was found through conversations with a DineSafe Quality Assurance staff that understaffing issues have resulted in certain low-risk food-establishments (e.g. convenience stores) skipping up to two years of inspections. Acknowledging this and other shortcomings in data methodology, this analysis will focus on surface-level findings.

#### This notebook will be split in two key parts: (1) [Methodology](#analysis) and (2) [Analysis](#analysis).


--- 



# Part I. Methodology <a class ="anchor" id = "methodology"></a>

## 1.1 Data Origin <a class ="anchor" id = "data_origin"></a>

Data was downloaded from the City of Toronto's [Open Data portal](https://www.toronto.ca/city-government/data-research-maps/open-data/) as of December 12th, 2018. However, this data only dates back two years. To extend our horizon, we accessed  the Web Archive (thanks [Twitter!](https://twitter.com/SherylsCrush/status/1068618721650982914)) and obtained DineSafe data since July 2011<sup>1</sup>. Since half of 2011 was missing, we decided to remove this year and start with a complete set of 2012 data. We acknowledge that the last two weeks or so of 2018 data are missing from our study, but we can also anticipate this will have minimal impact on our exploratory analysis.

#### Preliminary Cleaning
Because XML is gross, we formatted the raw data into a readable DataFrame. 

Original data showed the exact inspection date. As we only care about year, we converted that column to show only year.

Depending on their 'safety hazard' classification, establishments are inspected between one and three times a year. This would result in duplicates, whereas we only want establishments to appear maximum of once per year (to indicate existence). So as part of the preliminary cleaning process, we removed duplicated instances of **Establishment_IDs** per year.

#### Geocoding
Interestingly, only the December 2018 dataset had Latitude and Longitude attributes. We geocoded the remaining address using a combination of matching with available addresses, matching with [One Address Repository](https://www.toronto.ca/city-government/data-research-maps/open-data/open-data-catalogue/#f71a13c4-fb51-6116-57b7-1f51a8190585) addresses, and using Google's Geocoding API.

More cleaning/processing needs to be done but below is the initial dataset we have to work with:


```python
# Import libraries
from collections import Counter
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

from mapboxgl.utils import *
from mapboxgl.viz import *

# Set display options
pd.set_option("display.max_rows", 500)
pd.set_option('display.max_columns', 100)
pd.set_option("display.max_colwidth", 200)

# Load pre-processed Dinesafe Dataframe:
with open('Data/preprocessed_ds.csv','r') as f:
    ds = pd.read_csv(f, index_col = None)

print("We have {:,} rows of data and {:,} unique Establishment_IDs.".format(len(ds), len(ds['ESTABLISHMENT_ID'].unique())))
ds[:20]
```

    We have 99,587 rows of data and 27,400 unique Establishment_IDs.
    




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Year</th>
      <th>ESTABLISHMENTTYPE</th>
      <th>ESTABLISHMENT_ADDRESS</th>
      <th>ESTABLISHMENT_ID</th>
      <th>ESTABLISHMENT_NAME</th>
      <th>INSPECTION_ID</th>
      <th>LATITUDE</th>
      <th>LONGITUDE</th>
      <th>risk_level</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2012</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>102778249</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2013</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>103071804</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2014</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>103385541</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2015</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>103460005</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2016</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>103873791</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>5</th>
      <td>2017</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>103985362</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>6</th>
      <td>2018</td>
      <td>Restaurant</td>
      <td>5641 STEELES AVE E</td>
      <td>10185207</td>
      <td>TIM HORTONS</td>
      <td>104152523</td>
      <td>43.832617</td>
      <td>-79.266886</td>
      <td>2</td>
    </tr>
    <tr>
      <th>7</th>
      <td>2012</td>
      <td>Institutional Food Service</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>102877974</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>8</th>
      <td>2013</td>
      <td>Retirement Homes(Licensed)</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>102975081</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>9</th>
      <td>2014</td>
      <td>Retirement Homes(Licensed)</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>103214685</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>10</th>
      <td>2015</td>
      <td>Retirement Homes(Licensed)</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>103558527</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>11</th>
      <td>2016</td>
      <td>Retirement Homes(Licensed)</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>103816750</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>12</th>
      <td>2017</td>
      <td>Retirement Homes(Licensed)</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>104116194</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>13</th>
      <td>2018</td>
      <td>Retirement Homes(Licensed)</td>
      <td>15 BARBERRY PL</td>
      <td>10185216</td>
      <td>AMICA AT BAYVIEW</td>
      <td>104326543</td>
      <td>43.766412</td>
      <td>-79.384040</td>
      <td>3</td>
    </tr>
    <tr>
      <th>14</th>
      <td>2013</td>
      <td>Food Store (Convenience / Variety)</td>
      <td>2811 BATHURST ST</td>
      <td>10185464</td>
      <td>GLEN EDEN FRUIT MARKET</td>
      <td>102998706</td>
      <td>43.713267</td>
      <td>-79.428073</td>
      <td>1</td>
    </tr>
    <tr>
      <th>15</th>
      <td>2014</td>
      <td>Food Store (Convenience / Variety)</td>
      <td>2811 BATHURST ST</td>
      <td>10185464</td>
      <td>GLEN EDEN FRUIT MARKET</td>
      <td>103369478</td>
      <td>43.713267</td>
      <td>-79.428192</td>
      <td>1</td>
    </tr>
    <tr>
      <th>16</th>
      <td>2016</td>
      <td>Food Store (Convenience / Variety)</td>
      <td>2811 BATHURST ST</td>
      <td>10185464</td>
      <td>GLEN EDEN FRUIT MARKET</td>
      <td>103760900</td>
      <td>43.713267</td>
      <td>-79.428073</td>
      <td>1</td>
    </tr>
    <tr>
      <th>17</th>
      <td>2018</td>
      <td>Food Store (Convenience / Variety)</td>
      <td>2811 BATHURST ST</td>
      <td>10185464</td>
      <td>GLEN EDEN FRUIT MARKET</td>
      <td>104127241</td>
      <td>43.713267</td>
      <td>-79.428192</td>
      <td>1</td>
    </tr>
    <tr>
      <th>18</th>
      <td>2012</td>
      <td>Food Take Out</td>
      <td>504 BLOOR ST W</td>
      <td>10185511</td>
      <td>GHAZELE</td>
      <td>102814731</td>
      <td>43.665680</td>
      <td>-79.410535</td>
      <td>3</td>
    </tr>
    <tr>
      <th>19</th>
      <td>2013</td>
      <td>Food Take Out</td>
      <td>504 BLOOR ST W</td>
      <td>10185511</td>
      <td>GHAZALE</td>
      <td>102995216</td>
      <td>43.665680</td>
      <td>-79.410535</td>
      <td>3</td>
    </tr>
  </tbody>
</table>
</div>




```python
ds.info()
```

    <class 'pandas.core.frame.DataFrame'>
    RangeIndex: 99587 entries, 0 to 99586
    Data columns (total 9 columns):
    Year                     99587 non-null int64
    ESTABLISHMENTTYPE        99587 non-null object
    ESTABLISHMENT_ADDRESS    99587 non-null object
    ESTABLISHMENT_ID         99587 non-null int64
    ESTABLISHMENT_NAME       99587 non-null object
    INSPECTION_ID            99587 non-null int64
    LATITUDE                 99587 non-null float64
    LONGITUDE                99587 non-null float64
    risk_level               99587 non-null int64
    dtypes: float64(2), int64(4), object(3)
    memory usage: 6.8+ MB
    

We have 99,587 rows of data. Ultimately, we want to transform this data into a DataFrame that shows when establishments existed each year, from which we can visualize 'change' year over year. But first we need to clean out certain datapoints which may add noise from our study. As we can see from the graph below, the majority of food establishments are restaurants and take-out spots.

#### Caveats
- While most addresses are for standalone establishments, it's important to highlight that several are retail plazas. As such, several establishments could exist simultaneously, disallowing us from pursuing the easy route of simply grouping our data by address to show turnover. Doing so would cause one establishment to incorrectly represent a plaza that may actually have 10 establishments. To account for this, we first group by **Establishment_ID** in a dataframe then, from that dataframe, group by address (explained in the following 'Cleaning' section).


- We noticed that some establishments would skip a year of inspection - despite the requirement for having a minimum of one inspection per year. Per conversations with DineSafe staff, this was due to staffing issues and a backlog in inspections. This was more likely for low risk establishments like Gateway Newsstands. Standards were higher for high risk level establishments (e.g. those that handled raw meats). 

 To address this, we built a function that would selectively fill in gaps in years (in separate notebook on [GitHub](https://github.com/fabhlc/DineSafe_Gentrification)).


- Although an address may have the same name through the years, we also notice sometimes the **Establishment_ID** would change. DineSafe staff noted that any change in establishment ID is a result of change in ownership. In certain cases, this could mean the establishment continued operations under new management (owner). We count this as a 'change'. Notably, renovations do not result in a change in **Establishment_ID**. 


```python
type_labels = ds['ESTABLISHMENTTYPE'].value_counts().keys()
fig, ax = plt.subplots(figsize = (20, 16))
ax.barh(width = ds['ESTABLISHMENTTYPE'].value_counts(), y = range(len(type_labels)))
plt.yticks(range(len(type_labels)), type_labels, fontsize = 14)
plt.xlabel("Occurrences", size = 18)
plt.xticks(size = 18)
plt.title('Histogram of Establishment Types', fontsize = 20)
ax.invert_yaxis()
plt.grid()
plt.show()
```


![png](output_4_0.png)


To keep this notebook concise, you can find detailed steps for data cleaning and transformation (including criteria for keeping certain establishment types) in a separate notebook in my [GitHub](https://github.com/fabhlc/DineSafe).

## 1.2 Cleaning <a class ="anchor" id = "cleaning"></a>

Firstly, we rid the data of establishment types that may not necessarily aid our analysis. This includes things like Retirement Homes wth catering services, mobile hot dog carts, hospitals, and cruise boats. Food court vendors were also removed from our data as we wanted to concentrate the focus on standalone restaurants. This leaves us at 86,079 rows.

Next we remove data from large venues and stadiums like *Exhibition Place, Rogers Centre*, and *Downsview Flea Market*. Similarly, these often have different characteristics that impact turnover rates differently compared to 'stand-alone' addresses we're looking at. We are now at 83,520 rows.

Minor cleanups were needed along the way, such as correcting "5078 DUNDAS ST E" to "5078 DUNDAS ST W", replacing "&" with "AND" for consistency, and replacing apostrophes.

#### Key DataFrames
From our original data, we derive two key dataframes:
1. **turnover**: Turnover by Establishment ID
2. **turnover_address**: Turnover by address and their associated neighbourhoods. (Neighbourhood association executed by QGIS)

## 1.3 Feature Engineering <a class ="anchor" id = "feature_engineering"></a>
#### Dataframe 1: turnover 
This first dataframe was built to show in which years establishments existed. A key feature engineered out of the data and shown as columns in this dataframe is **yearly change**. This binary data indicates whether an establishment continued its existence or non-existence from year to year. For example, if an establishment existed in 2012 but ceased in 2013, this would be indicated by a "1" in change_2013. 

As previously mentioned, certain establishments show gaps in existence (as a very real consequence of staffing issues). We built a function to fill in these gaps. Essentially, where there were more than two 'changes' in the six 'change' columns, the function would convert these existence gaps from 0 to 1. 


#### Dataframe 2: turnover_address
From the first dataframe, we grouped the data by address to shift the focus on addresses, creating the second dataframe, **'turnover_address'**. Additional features were engineered out of this to measure change:
- ** Total Turnover**: This is the total number of changes at this address (e.g. sum of all closures and openings).
- ** Annual Turnover** : Total turnover divided by the number of years (six).
- ** Max Units **: Maximum number of units existing at a single address in any given year between 2012 and 2018. (Ideally we would have data on the number of retail units at an address. This assumption would be the next best thing).
- ** Change Index **: annual turnover divided by max units (to account for different weights of addresses with multiple establishment units). 
- ** Multi Units **: Binary for if there is more than one establishment at this address.

#### Change Index
Change Index values typically span from 0 to 1, with zero indicating no change throughout the six year period. For example, a Change Index of 0.5 would mean an address has a 50% chance of 'changing' each year, whether by closure or new opening. The city-wide average change index of 0.2114 indicates food-establishment addresses have a 21% chance of turning over each year.

Only 11 addresses (0.09% of addresses) exceed 1.0 and range from 1.33 to 1.67. This is because openings and closings in establishments account for 'one' change each, so if an establishment closes by the end of 2014 and another opens in its place in 2015, **change_2015** would have a value of 2.


Let's load these three dataframes into this notebook.


```python
with open('Data/turnover_id.csv', 'r') as f:
    turnover = pd.read_csv(f, index_col = None)
turnover[:15]
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>name</th>
      <th>address</th>
      <th>exist_2012</th>
      <th>exist_2013</th>
      <th>exist_2014</th>
      <th>exist_2015</th>
      <th>exist_2016</th>
      <th>exist_2017</th>
      <th>exist_2018</th>
      <th>change_2013</th>
      <th>change_2014</th>
      <th>change_2015</th>
      <th>change_2016</th>
      <th>change_2017</th>
      <th>change_2018</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>"FOOD APPEAL.CA INC."</td>
      <td>222 ISLINGTON AVE</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>1</th>
      <td>(FAMOUS PLAYERS )CINEPLEX ENTERTAINMENT</td>
      <td>2190 YONGE ST</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>2</th>
      <td>0109 DESSERT + CHOCOLATE</td>
      <td>2190 MCNICOLL AVE</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1 PLUS 1 PIZZA</td>
      <td>361 OAKWOOD AVE</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1 PLUS 2 PIZZA AND WING</td>
      <td>3260 DUNDAS ST W</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>5</th>
      <td>1 PLUS 2 PIZZA AND WINGS</td>
      <td>1540 DANFORTH AVE</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1 PLUS 3 PIZZA AND WINGS</td>
      <td>1798 JANE ST</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>7</th>
      <td>100 HUMBER COLLEGE CAFE</td>
      <td>100 HUMBER COLLEGE BLVD</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>8</th>
      <td>100 KM FOODS INC.</td>
      <td>4478 CHESSWOOD DR</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>9</th>
      <td>100 POR SIENTO SALVADORENO</td>
      <td>404 OLD WESTON RD</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>10</th>
      <td>100% KOREAN</td>
      <td>109 RAVEL RD</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>11</th>
      <td>100% KOREAN</td>
      <td>4779 STEELES AVE E</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
    </tr>
    <tr>
      <th>12</th>
      <td>100% SALVA DO RENO</td>
      <td>612 TRETHEWEY DR</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr>
      <th>13</th>
      <td>1000 VARIETY</td>
      <td>1000 PAPE AVE</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
    </tr>
    <tr>
      <th>14</th>
      <td>10733440 CANADA INC</td>
      <td>30 THORNCLIFFE PARK DR</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
    </tr>
  </tbody>
</table>
</div>




```python
with open('Data/turnover_address (QGIS_withneighbourhoods).csv', 'r') as f:
    turnover_address = pd.read_csv(f, index_col = None)
    
# Create a more compact dataframe without the change/exist columns
turnover_address_1 = pd.concat((turnover_address.sort_values('change_index', ascending = False).iloc[:,:1],
                                turnover_address.sort_values('change_index', ascending = False).iloc[:,14:]), axis = 1)

turnover_address[:15]
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>address</th>
      <th>exist_2012</th>
      <th>exist_2013</th>
      <th>exist_2014</th>
      <th>exist_2015</th>
      <th>exist_2016</th>
      <th>exist_2017</th>
      <th>exist_2018</th>
      <th>change_2013</th>
      <th>change_2014</th>
      <th>change_2015</th>
      <th>change_2016</th>
      <th>change_2017</th>
      <th>change_2018</th>
      <th>total_turnover</th>
      <th>annual_turnover</th>
      <th>max_units</th>
      <th>change_index</th>
      <th>multi_units</th>
      <th>LATITUDE</th>
      <th>LONGITUDE</th>
      <th>AREA_NAME</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1 BALMORAL AVE</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>2</td>
      <td>0.33333</td>
      <td>1</td>
      <td>0.33333</td>
      <td>0</td>
      <td>43.68554</td>
      <td>-79.39342</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>1</th>
      <td>11 ST CLAIR AVE W</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0.00000</td>
      <td>1</td>
      <td>0.00000</td>
      <td>0</td>
      <td>43.68777</td>
      <td>-79.39457</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>2</th>
      <td>111 ST CLAIR AVE W</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0.16667</td>
      <td>1</td>
      <td>0.16667</td>
      <td>0</td>
      <td>43.68665</td>
      <td>-79.39951</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1208 YONGE ST</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0.16667</td>
      <td>1</td>
      <td>0.16667</td>
      <td>0</td>
      <td>43.68172</td>
      <td>-79.39171</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1210 YONGE ST</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>2</td>
      <td>0.33333</td>
      <td>1</td>
      <td>0.33333</td>
      <td>0</td>
      <td>43.68175</td>
      <td>-79.39174</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>5</th>
      <td>1234 YONGE ST</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>2</td>
      <td>0.33333</td>
      <td>1</td>
      <td>0.33333</td>
      <td>0</td>
      <td>43.68258</td>
      <td>-79.39211</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1240 YONGE ST</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>2</td>
      <td>0.33333</td>
      <td>1</td>
      <td>0.33333</td>
      <td>0</td>
      <td>43.68292</td>
      <td>-79.39220</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1246 YONGE ST</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0.16667</td>
      <td>1</td>
      <td>0.16667</td>
      <td>0</td>
      <td>43.68304</td>
      <td>-79.39234</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1250 YONGE ST</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0.16667</td>
      <td>1</td>
      <td>0.16667</td>
      <td>0</td>
      <td>43.68312</td>
      <td>-79.39246</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>9</th>
      <td>1272 YONGE ST</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>0.16667</td>
      <td>1</td>
      <td>0.16667</td>
      <td>0</td>
      <td>43.68357</td>
      <td>-79.39248</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1274 YONGE ST</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0.16667</td>
      <td>1</td>
      <td>0.16667</td>
      <td>0</td>
      <td>43.68358</td>
      <td>-79.39250</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>11</th>
      <td>1280 YONGE ST</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>2</td>
      <td>0.33333</td>
      <td>1</td>
      <td>0.33333</td>
      <td>0</td>
      <td>43.68368</td>
      <td>-79.39253</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>12</th>
      <td>13 ST CLAIR AVE W</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0.00000</td>
      <td>1</td>
      <td>0.00000</td>
      <td>0</td>
      <td>43.68775</td>
      <td>-79.39467</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>13</th>
      <td>1366 YONGE ST</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0.16667</td>
      <td>2</td>
      <td>0.08333</td>
      <td>0</td>
      <td>43.68604</td>
      <td>-79.39359</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
    <tr>
      <th>14</th>
      <td>1378 YONGE ST</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>2</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>0.00000</td>
      <td>2</td>
      <td>0.00000</td>
      <td>0</td>
      <td>43.68625</td>
      <td>-79.39356</td>
      <td>Yonge-St.Clair (97)</td>
    </tr>
  </tbody>
</table>
</div>



Out of these dataframes, we'll create a reference dataframe that will quickly show us the establishments associated with each address. This will come in handy for mapping later.


```python
# Create a reference table for establishments and addresses.

turnover['first_exist'] = ['201'+str(list(i).index(1)) for i in turnover.values]
ests = turnover.groupby('address')['name'].apply(list)
yrs = turnover.groupby('address')['first_exist'].apply(list)
zipped = list(zip(ests, yrs))

est_by_add = pd.DataFrame(ests).join(pd.DataFrame(yrs))
est_by_add.reset_index(inplace=True)

# Clean fields to make it more readable. "Esta." must be clean as it will feed into the map labels later.
est_by_add['Esta.'] = [list(zip(i,j)) for (i,j) in zipped] 
est_by_add['name'] = [", ".join(i) for i in est_by_add['name']]
est_by_add['first_exist'] = [", ".join(i) for i in est_by_add['first_exist']]
est_by_add['Esta.'] = [", ".join(["{} ({})".format(j[0].title(), j[1]) for j in i]) for i in est_by_add['Esta.']]

est_by_add[:15]
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>address</th>
      <th>name</th>
      <th>first_exist</th>
      <th>Esta.</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>1 ADELAIDE ST E</td>
      <td>CRAFT BEER MARKET RESTAURANT AND BAR, INTERNATIONAL NEWS, STARBUCKS</td>
      <td>2017, 2012, 2012</td>
      <td>Craft Beer Market Restaurant And Bar (2017), International News (2012), Starbucks (2012)</td>
    </tr>
    <tr>
      <th>1</th>
      <td>1 AUSTIN TER</td>
      <td>BLUE BLOOD STEAKHOUSE, CASA LOMA, SIR HENRYS CAFE</td>
      <td>2017, 2012, 2012</td>
      <td>Blue Blood Steakhouse (2017), Casa Loma (2012), Sir Henrys Cafe (2012)</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1 AVONDALE AVE</td>
      <td>PIZZA NOVA, STARBUCKS COFFEE CO</td>
      <td>2012, 2012</td>
      <td>Pizza Nova (2012), Starbucks Coffee Co (2012)</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1 BALDWIN ST</td>
      <td>MO RAMYUN, YAKITORI BAR AND SEOUL FOOD COMPANY</td>
      <td>2015, 2013</td>
      <td>Mo Ramyun (2015), Yakitori Bar And Seoul Food Company (2013)</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1 BALMORAL AVE</td>
      <td>BARNSTEINERS, JOHN AND SONS OYSTER HOUSE</td>
      <td>2015, 2012</td>
      <td>Barnsteiners (2015), John And Sons Oyster House (2012)</td>
    </tr>
    <tr>
      <th>5</th>
      <td>1 BAXTER ST</td>
      <td>EL TENEDOR, GOLDEN MINT, THE MINT COFFEE AND TEA CO.</td>
      <td>2016, 2013, 2015</td>
      <td>El Tenedor (2016), Golden Mint (2013), The Mint Coffee And Tea Co. (2015)</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1 BEDFORD RD</td>
      <td>STARBUCKS COFFEE</td>
      <td>2012</td>
      <td>Starbucks Coffee (2012)</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1 BENVENUTO PL</td>
      <td>SCARAMOUCHE</td>
      <td>2012</td>
      <td>Scaramouche (2012)</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1 BESTOBELL RD</td>
      <td>FANTIS FOODS OF CANADA LTD.</td>
      <td>2012</td>
      <td>Fantis Foods Of Canada Ltd. (2012)</td>
    </tr>
    <tr>
      <th>9</th>
      <td>1 BLOOR ST E</td>
      <td>STARBUCKS</td>
      <td>2018</td>
      <td>Starbucks (2018)</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1 BLUE GOOSE ST</td>
      <td>THE BLUE GOOSE TAVERN</td>
      <td>2013</td>
      <td>The Blue Goose Tavern (2013)</td>
    </tr>
    <tr>
      <th>11</th>
      <td>1 BONIS AVE</td>
      <td>GUSTO PIZZA</td>
      <td>2013</td>
      <td>Gusto Pizza (2013)</td>
    </tr>
    <tr>
      <th>12</th>
      <td>1 BRIDGEPOINT DR</td>
      <td>FOOD HALL</td>
      <td>2016</td>
      <td>Food Hall (2016)</td>
    </tr>
    <tr>
      <th>13</th>
      <td>1 BRIMLEY RD S</td>
      <td>CATHEDRAL BLUFFS YACHT CLUB RESTAURANT</td>
      <td>2017</td>
      <td>Cathedral Bluffs Yacht Club Restaurant (2017)</td>
    </tr>
    <tr>
      <th>14</th>
      <td>1 BYNG AVE</td>
      <td>BIG BEEF BOWL, IRAJ MARKET, SANG JI FRIED BAO</td>
      <td>2016, 2014, 2018</td>
      <td>Big Beef Bowl (2016), Iraj Market (2014), Sang Ji Fried Bao (2018)</td>
    </tr>
  </tbody>
</table>
</div>



---


# Part II. Analysis <a class ="anchor" id = "analysis"></a>

Let's take a look at the geographical distribution of food-establishments. Below is a heatmap showing where food establishment addresses are concentrated (weighted by **max_units**) over a basemap comprising boundaries for the City of Toronto's official 140 neighbourhoods (numbered).

![FINAL%20Dinesafe_Geospatial%20distribution.png](attachment:FINAL%20Dinesafe_Geospatial%20distribution.png)

A closer look at the concentration downtown unsurprisingly shows concentrations along main arterials, particularly those with subway lines. Kensington Market appears to have one of the densest clusters of food establishments, followed by food courts at 10 Dundas across from Dundas Square, and the food courts along the PATH.

![FINAL%20Dinesafe_Geospatial%20distribution%20ZOOM.png](attachment:FINAL%20Dinesafe_Geospatial%20distribution%20ZOOM.png)

## 2.1 Closures <a class ="anchor" id = "closures"></a>
While heatmaps are great for showing geospatial concentrations, they are less suited in helping us visualize the magnitude of turnovers and closures as they will get conflated with geospatially-concentrated retail areas. Looking at closures may provide us with a better sense of where businesses are being pushed out. Let's look at establishments that existed in 2012 but ceased to exist 2018 onwards (i.e. closed anytime between 2013 and 2017).


```python
closures = turnover[(turnover['exist_2012'] == 1) & (turnover['exist_2018'] == 0)]
closures = closures.join(turnover_address[['address', 'LATITUDE','LONGITUDE', 'AREA_NAME']].set_index('address'), 
                         on = 'address', how = 'left')

# How many closed in 2014? In 2015? 2016?
print("{:,} food-establishments closed in 2013.\n{:,} closed in 2014.\n{:,} closed in 2015.\n{:,} closed in 2016.".format(sum(closures['change_2013']),
                                                                              sum(closures['change_2014']),
                                                                             sum(closures['change_2015']),
                                                                             sum(closures['change_2016'])))
```

    557 food-establishments closed in 2013.
    1,481 closed in 2014.
    717 closed in 2015.
    728 closed in 2016.
    

There was a shockingly high number of restaurant closures in 2014 - more than double that of adjacent years. A quick Google dive shows us the prolific Mr Greenjeans was among the list of casualties, after a long 34-year run in the Eaton Centre.

![mr-greenjeans-restaurant.jpg](attachment:mr-greenjeans-restaurant.jpg)
<div style="text-align: center"><i> Mr. Greenjeans Restaurant. (RIP. 1980 to 2014) Source: TripAdvisor </i> </div>

The bulk of 2014's closures were downtown in the Bay Street Corridor neighbourhood (see table below). Most of these closures were fast-food type of businesses, like Tim Hortons, grab-n-go sushi shops, and Gateway Newsstands. Keeping in mind this area has a dense clustering of food establishments catering to office employees as well as high rents for small spaces, high turnover would make sense. However, I have yet to find out exactly why there was a spike in 2014 (industry insights welcome!).


```python
closures_15 = closures.groupby('AREA_NAME').sum()[['change_2013','change_2014',
                                                   'change_2015','change_2016']].sort_values('change_2014', ascending=False).iloc[:15,:]
closures_15.columns = ['Closed_2013', 'Closed_2014', 'Closed_2015', 'Closed_2016']
print("Top 15 Neighbourhoods with highest no. of closures in 2014:")
closures_15
```

    Top 15 Neighbourhoods with highest no. of closures in 2014:
    




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Closed_2013</th>
      <th>Closed_2014</th>
      <th>Closed_2015</th>
      <th>Closed_2016</th>
    </tr>
    <tr>
      <th>AREA_NAME</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Bay Street Corridor (76)</th>
      <td>22</td>
      <td>115</td>
      <td>38</td>
      <td>25</td>
    </tr>
    <tr>
      <th>Waterfront Communities-The Island (77)</th>
      <td>11</td>
      <td>69</td>
      <td>36</td>
      <td>37</td>
    </tr>
    <tr>
      <th>Kensington-Chinatown (78)</th>
      <td>11</td>
      <td>67</td>
      <td>26</td>
      <td>34</td>
    </tr>
    <tr>
      <th>Annex (95)</th>
      <td>9</td>
      <td>54</td>
      <td>25</td>
      <td>23</td>
    </tr>
    <tr>
      <th>Church-Yonge Corridor (75)</th>
      <td>13</td>
      <td>53</td>
      <td>35</td>
      <td>26</td>
    </tr>
    <tr>
      <th>Trinity-Bellwoods (81)</th>
      <td>8</td>
      <td>40</td>
      <td>15</td>
      <td>16</td>
    </tr>
    <tr>
      <th>Milliken (130)</th>
      <td>4</td>
      <td>38</td>
      <td>10</td>
      <td>23</td>
    </tr>
    <tr>
      <th>Islington-City Centre West (14)</th>
      <td>2</td>
      <td>33</td>
      <td>13</td>
      <td>16</td>
    </tr>
    <tr>
      <th>Dovercourt-Wallace Emerson-Junction (93)</th>
      <td>16</td>
      <td>24</td>
      <td>7</td>
      <td>32</td>
    </tr>
    <tr>
      <th>Palmerston-Little Italy (80)</th>
      <td>11</td>
      <td>23</td>
      <td>12</td>
      <td>11</td>
    </tr>
    <tr>
      <th>York University Heights (27)</th>
      <td>38</td>
      <td>22</td>
      <td>19</td>
      <td>9</td>
    </tr>
    <tr>
      <th>University (79)</th>
      <td>0</td>
      <td>22</td>
      <td>9</td>
      <td>7</td>
    </tr>
    <tr>
      <th>Yorkdale-Glen Park (31)</th>
      <td>9</td>
      <td>21</td>
      <td>4</td>
      <td>2</td>
    </tr>
    <tr>
      <th>Roncesvalles (86)</th>
      <td>2</td>
      <td>21</td>
      <td>10</td>
      <td>12</td>
    </tr>
    <tr>
      <th>South Riverdale (70)</th>
      <td>32</td>
      <td>20</td>
      <td>20</td>
      <td>12</td>
    </tr>
  </tbody>
</table>
</div>



## 2.2 Turnover  <a class ="anchor" id = "turnover"></a>
As previously mentioned, turnover is defined as total number of changes divided by six (2012-2018), divided by the maximum number of units at an address (most often one, but may be more for retail plazas). 


```python
city_ci = turnover_address['annual_turnover'].sum() / turnover_address['max_units'].sum()
print("The average annual turnover by address (∑ annual turnover / ∑ max units) is {:.4f}.".format(city_ci))
```

    The average annual turnover by address (∑ annual turnover / ∑ max units) is 0.2114.
    

This means the average food-establishment in Toronto experiences a change approximately once every five years (0.21 times per year).

### 2.2.1 Turnover by Former Municipality  <a class ="anchor" id = "turnover_by_fm"></a>
We'll look at the average Change Index of each former municipality to assess if there are noticeable difference at this geographical level. Attributing each address to a former municipality was already done in QGIS, so we'll just load that file and perform a *groupby* aggregation.


```python
# Open a file that attributes turnover addresses to former municipality (executed in QGIS)
with open('Data/QGIS/QGIS_turnoveraddress_x_formermuni.csv', 'r') as f:
    former_muni = pd.read_csv(f)
turnover_formermuni = former_muni[['AREA_NAME','annual_turnover']].groupby('AREA_NAME').sum()/former_muni[['AREA_NAME','max_units']].groupby('AREA_NAME').sum().values
turnover_formermuni.columns = ['Average Change Index']
turnover_formermuni
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>Average Change Index</th>
    </tr>
    <tr>
      <th>AREA_NAME</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>EAST YORK</th>
      <td>0.210815</td>
    </tr>
    <tr>
      <th>ETOBICOKE</th>
      <td>0.193966</td>
    </tr>
    <tr>
      <th>NORTH YORK</th>
      <td>0.212582</td>
    </tr>
    <tr>
      <th>SCARBOROUGH</th>
      <td>0.239306</td>
    </tr>
    <tr>
      <th>TORONTO</th>
      <td>0.204877</td>
    </tr>
    <tr>
      <th>YORK</th>
      <td>0.223846</td>
    </tr>
  </tbody>
</table>
</div>



Scarborough has the highest rate of turnover on average while Etobicoke seems to be the most stable. When we move a level down to the neighbourhood level, we see that less aggregated data at this level shows that individual neighbourhoods in Scarborough have a noticeably higher rate of turnover than the rest of the city. The map below shows how the Change Index of each neighbourhood compares to the city-average. More mellow colours indicate turnover closer to the city average Change Index of 0.2114, while more red pigments indicate neighbourhoods with higher-than-average turnover and conversely for blue pigments.

<img src="FINAL%20Change_Index%20Diff%20from%20City%20Avg%20by%20Neighbourhood.png"/>

### Scarborough
It's difficult to overlook these trends in Scarborough so it's worth mentioning a few points about this ethnically and socioeconomically diverse suburb of Toronto. There are several obstacles to operating a successful food establishment in Scarborough that make it different from more central parts of Toronto:
- Scarborough has lower spending power than the rest of Toronto (\$79,000 average household income in Scarborough vs. \$103,000 across Toronto as of 2015)<sup>2</sup>. This translates to less disposable income for eating out - a pricing consideration for the area's food-establishments.
- Scarborough's population comprises almost two-thirds immigrants and non-permanent residents<sup>2</sup>. Although more recent immigrants have much higher spending power<sup>3</sup>, earlier immigrants are generally more prudent and financially conservative, especially in the time following immigration. These cultural and socioeconomic factors push towards them being less likely to eat out on a regular basis. On the other hand, time passed since their immigration as well as today's more diverse food scene reflecting their homeland's cuisines may be shifting inclinations.
- Anecdotally, successful restaurants opened by immigrants from the 90s are shutting down as owners are retiring and their adult children don't want to take over.
- Millennials are influenced by social status and the "instagrammability" of restaurants they eat out in - which is often downtown.
- The suburban built form of Scarborough and street-facing-parking orientation of retail plazas inhibit the visibility of restaurants. This makes it harder for passersby to notice and discover restaurants. While marketers may help a business overcome this with a strong social media presence, the older generation of restaurateurs may struggle to attract a steady stream of new customers.

Given the above socioeconomic, cultural, and built-form factors, and assuming a similar set of differences in the other boroughs of Etobicoke and North York, it is difficult to paint Toronto and Scarborough with the same brush. A deeper dive into Scarborough's food scene may be worth pursuing, but it would be out of scope for this notebook.  

#### Side Note:
That being said, it should be noted that when we google 'restaurant closure toronto', we get detailed articles mourning the loss of neighbourhood fixtures in more downtown-centric (and predominantly White) neighbourhoods like Leslieville ([one](https://www.blogto.com/eat_drink/2018/02/lil-baci-italian-restaurant-closing-toronto/) and [two](https://nowtoronto.com/food-and-drink/food/saturday-dinette-closed-toronto/)), [Parkdale](https://www.cbc.ca/news/canada/toronto/we-re-being-squeezed-out-locals-try-to-save-parkdale-restaurant-amid-gentrification-worry-1.4359744), [Little Italy](https://torontolife.com/food/restaurants/grace-restaurant-closed/), and [Chinatown](https://tvo.org/article/current-affairs/why-the-closure-of-a-restaurant-can-feel-like-a-death) (frequenters of Toronto's Chinatown today more closely resemble a United Colors of Benetton ad). However, searches for 'restaurant closure scarborough' yield nothing, despite anecdotes of established local neighbourhood favourites closing. A question worthwhile asking may be "where are all the obituaries in mainstream media for Scarborough's (and Etobicoke's) established favourites? How will their contributions and service as a gathering place for the local community be documented?"

###  2.2.2 Turnover by Neighbourhood  <a class ="anchor" id = "turnover_by_n"></a>
Attributing each datapoint to their corresponding neighbourhood, we'll create a dataframe that shows the number of addresses within each neighbourhood in each **change_index** bin, as well as the overall average **change_index** and the difference to the city-wide average. The table below is sorted by total number of addresses with a **change_index** of over 0.50 (i.e. turnover once every two years on average).

 _Methodology note: we segregate the data using absolute numbers as opposed to proportion (i.e. % of neighbourhood with 0.50 change index) as we've noticed certain hotspots are diluted at the neighbourhood level by more stable addresses._


```python
# Create Series
n_over75 = turnover_address[turnover_address['change_index'] >= 0.75].groupby('AREA_NAME')['address'].count()
n_50to75 = turnover_address[(turnover_address['change_index'] >= 0.50) & (turnover_address['change_index'] < 0.75)].groupby('AREA_NAME')['address'].count()
n_20to50 = turnover_address[(turnover_address['change_index'] >= 0.20) & (turnover_address['change_index'] < 0.50)].groupby('AREA_NAME')['address'].count()
n_under20 = turnover_address[turnover_address['change_index'] < 0.20].groupby('AREA_NAME')['address'].count()# Note that the mean change_index by address is 0.208, thus we chose 0.20 to be the threshold for lower.
n_total = turnover_address.groupby('AREA_NAME')['AREA_NAME'].count()

# Combine series into dataframe
change_bins = pd.concat([n_over75, n_50to75, n_20to50, n_under20, n_total], axis = 1).fillna(0)
change_bins.columns = ['over 0.75', '0.50 to 0.75', '0.20 to 0.50', 'under 0.20', 'total']
change_bins['total over 0.50'] = change_bins['over 0.75'] + change_bins['0.50 to 0.75']

change_bins['avg_change_index'] = turnover_address.groupby('AREA_NAME')['annual_turnover'].sum()/turnover_address.groupby('AREA_NAME')['max_units'].sum()
change_bins['mean_diff'] = change_bins['avg_change_index'] - city_ci

print("Top 20 neighbourhoods with addresses with turnover over 0.50:")
change_bins.sort_values(['total over 0.50'], ascending = False).iloc[:20,:]
```

    Top 20 neighbourhoods with addresses with turnover over 0.50:
    




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>over 0.75</th>
      <th>0.50 to 0.75</th>
      <th>0.20 to 0.50</th>
      <th>under 0.20</th>
      <th>total</th>
      <th>total over 0.50</th>
      <th>avg_change_index</th>
      <th>mean_diff</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>Kensington-Chinatown (78)</th>
      <td>8.0</td>
      <td>50.0</td>
      <td>132</td>
      <td>282</td>
      <td>472</td>
      <td>58.0</td>
      <td>0.226550</td>
      <td>0.015107</td>
    </tr>
    <tr>
      <th>Trinity-Bellwoods (81)</th>
      <td>4.0</td>
      <td>35.0</td>
      <td>65</td>
      <td>204</td>
      <td>308</td>
      <td>39.0</td>
      <td>0.203759</td>
      <td>-0.007684</td>
    </tr>
    <tr>
      <th>Annex (95)</th>
      <td>2.0</td>
      <td>33.0</td>
      <td>71</td>
      <td>227</td>
      <td>333</td>
      <td>35.0</td>
      <td>0.186964</td>
      <td>-0.024480</td>
    </tr>
    <tr>
      <th>Church-Yonge Corridor (75)</th>
      <td>7.0</td>
      <td>26.0</td>
      <td>92</td>
      <td>273</td>
      <td>398</td>
      <td>33.0</td>
      <td>0.196518</td>
      <td>-0.014925</td>
    </tr>
    <tr>
      <th>Willowdale East (51)</th>
      <td>2.0</td>
      <td>28.0</td>
      <td>36</td>
      <td>75</td>
      <td>141</td>
      <td>30.0</td>
      <td>0.278049</td>
      <td>0.066606</td>
    </tr>
    <tr>
      <th>Waterfront Communities-The Island (77)</th>
      <td>0.0</td>
      <td>29.0</td>
      <td>124</td>
      <td>338</td>
      <td>491</td>
      <td>29.0</td>
      <td>0.184531</td>
      <td>-0.026912</td>
    </tr>
    <tr>
      <th>Bay Street Corridor (76)</th>
      <td>3.0</td>
      <td>26.0</td>
      <td>106</td>
      <td>236</td>
      <td>371</td>
      <td>29.0</td>
      <td>0.205962</td>
      <td>-0.005481</td>
    </tr>
    <tr>
      <th>South Riverdale (70)</th>
      <td>1.0</td>
      <td>28.0</td>
      <td>73</td>
      <td>189</td>
      <td>291</td>
      <td>29.0</td>
      <td>0.206897</td>
      <td>-0.004546</td>
    </tr>
    <tr>
      <th>Dovercourt-Wallace Emerson-Junction (93)</th>
      <td>4.0</td>
      <td>24.0</td>
      <td>54</td>
      <td>169</td>
      <td>251</td>
      <td>28.0</td>
      <td>0.205919</td>
      <td>-0.005525</td>
    </tr>
    <tr>
      <th>Palmerston-Little Italy (80)</th>
      <td>5.0</td>
      <td>21.0</td>
      <td>42</td>
      <td>111</td>
      <td>179</td>
      <td>26.0</td>
      <td>0.225000</td>
      <td>0.013557</td>
    </tr>
    <tr>
      <th>Danforth (66)</th>
      <td>3.0</td>
      <td>20.0</td>
      <td>40</td>
      <td>90</td>
      <td>153</td>
      <td>23.0</td>
      <td>0.225904</td>
      <td>0.014461</td>
    </tr>
    <tr>
      <th>Junction Area (90)</th>
      <td>2.0</td>
      <td>17.0</td>
      <td>46</td>
      <td>114</td>
      <td>179</td>
      <td>19.0</td>
      <td>0.202493</td>
      <td>-0.008951</td>
    </tr>
    <tr>
      <th>Woburn (137)</th>
      <td>2.0</td>
      <td>17.0</td>
      <td>67</td>
      <td>62</td>
      <td>148</td>
      <td>19.0</td>
      <td>0.278267</td>
      <td>0.066824</td>
    </tr>
    <tr>
      <th>Roncesvalles (86)</th>
      <td>2.0</td>
      <td>17.0</td>
      <td>64</td>
      <td>122</td>
      <td>205</td>
      <td>19.0</td>
      <td>0.207589</td>
      <td>-0.003854</td>
    </tr>
    <tr>
      <th>East End-Danforth (62)</th>
      <td>1.0</td>
      <td>17.0</td>
      <td>26</td>
      <td>89</td>
      <td>133</td>
      <td>18.0</td>
      <td>0.215131</td>
      <td>0.003688</td>
    </tr>
    <tr>
      <th>University (79)</th>
      <td>0.0</td>
      <td>18.0</td>
      <td>49</td>
      <td>95</td>
      <td>162</td>
      <td>18.0</td>
      <td>0.205446</td>
      <td>-0.005997</td>
    </tr>
    <tr>
      <th>North Riverdale (68)</th>
      <td>1.0</td>
      <td>17.0</td>
      <td>25</td>
      <td>65</td>
      <td>108</td>
      <td>18.0</td>
      <td>0.223164</td>
      <td>0.011721</td>
    </tr>
    <tr>
      <th>Little Portugal (84)</th>
      <td>1.0</td>
      <td>17.0</td>
      <td>35</td>
      <td>89</td>
      <td>142</td>
      <td>18.0</td>
      <td>0.204733</td>
      <td>-0.006710</td>
    </tr>
    <tr>
      <th>Yorkdale-Glen Park (31)</th>
      <td>2.0</td>
      <td>16.0</td>
      <td>33</td>
      <td>77</td>
      <td>128</td>
      <td>18.0</td>
      <td>0.223810</td>
      <td>0.012367</td>
    </tr>
    <tr>
      <th>Wexford/Maryvale (119)</th>
      <td>2.0</td>
      <td>16.0</td>
      <td>49</td>
      <td>108</td>
      <td>175</td>
      <td>18.0</td>
      <td>0.218147</td>
      <td>0.006704</td>
    </tr>
  </tbody>
</table>
</div>



The top four neighbourhoods come as no surprise to millennials familiar with the dynamic nature of trendy neighbourhoods like [West Queen West](https://www.vogue.com/slideshow/fifteen-coolest-street-style-neighborhoods) and Toronto's own gaybourhood.


### Kensington-Chinatown
Kensington Market-Chinatown has the highest number of addresses with a change index over 0.75. Let's take a quick look at where these eight are and what establishments were/are at these addresses.


```python
kensington = turnover_address[(turnover_address['change_index'] > 0.75) & (turnover_address['AREA_NAME'].str.contains('Kensington'))]\
.join(est_by_add.set_index('address'), on = 'address', how = 'left')

kensington = pd.concat([kensington.loc[:,'address'], kensington.loc[:,['total_turnover', 'annual_turnover', 
                                                 'max_units', 'change_index', 'Esta.']]], axis=1)
kensington
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>address</th>
      <th>total_turnover</th>
      <th>annual_turnover</th>
      <th>max_units</th>
      <th>change_index</th>
      <th>Esta.</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>5100</th>
      <td>280 AUGUSTA AVE</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>Aaa Oishi Katta (2012), Frosty Roll (2016), Pizza Pala (2014)</td>
    </tr>
    <tr>
      <th>5227</th>
      <td>409 COLLEGE ST</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>32 Chicken St (2018), Coffee Culture (2015), Thrive Organic Kitchen And Cafe (2017)</td>
    </tr>
    <tr>
      <th>5290</th>
      <td>478 DUNDAS ST W</td>
      <td>14</td>
      <td>2.33333</td>
      <td>3</td>
      <td>0.77778</td>
      <td>Buk Chang Dongsoon Tofu   Dundas (2013), Delicious Hey Noodles (2017), Doig Bar (Szechuan Hot Pot) (2013), Harmony Chinese Restaurant (2015), Hey Noodles (2016), Min Jiang Restaurant (2013), Nam C...</td>
    </tr>
    <tr>
      <th>5305</th>
      <td>494 DUNDAS ST W</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>Cai Die Xuan Food (2012), Cai Die Xuan Food   Lamb Kebab (2016), Modern Gifts And Variety (2014)</td>
    </tr>
    <tr>
      <th>5327</th>
      <td>536 QUEEN ST W</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>Away Kitchen And Cafe (2018), Constantinople Bakery And Coffee (2015), Death In Venice (2016)</td>
    </tr>
    <tr>
      <th>5340</th>
      <td>56 KENSINGTON AVE</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>Kensington Variety (2013)</td>
    </tr>
    <tr>
      <th>5342</th>
      <td>566 QUEEN ST W</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>Ace Gale Tap House (2015), Michaels Restaurant (2014), Michaels Restaurant And Deli (2012)</td>
    </tr>
    <tr>
      <th>5366</th>
      <td>68 WALES AVE</td>
      <td>5</td>
      <td>0.83333</td>
      <td>1</td>
      <td>0.83333</td>
      <td>Cafe Unwind (2013), Sweet Hart Kitchen (2017), The Strong One (2015)</td>
    </tr>
  </tbody>
</table>
</div>



#### "Invisible" Turnover 
Interestingly, despite a high turnover at **56 Kensington Avenue**, the only establishment that has existed at this location between 2012 and 2018 is Kensington Variety. Throughout this six-year period, Kensington Variety has kept its name, but possessed three different establishment IDs - i.e. changed at least three times. As previously mentioned, this may be attributable to change in ownership. While a deeper dive is required to verify this, it brings to our awareness the effects of "invisible" turnover - where a business storefront remains the same but the original owner(s) has been bought out. To anyone interested, it may be worthwhile perusing a licensing database to see where establishments are being pushed out 'invisibly'.

<img src="56%20Kensington%20Ave.jpg"/>
<div style="text-align: center"><i> We haven't confirmed if ownership has changed, but storefront renovations are certainly consistent with this theory. Source: Google Streetview </i> </div>

#### Note: Data Weakness
This example admittedly shines a light on a shortcoming in our data - 56 Kensington Avenue appears vacant in our data during certain years but this isn't true.  Understaffing issues are to blame, which create a wrinkle in the premise of "minimum once a year" inspections. On the other hand, only low-risk establishments like convenience stores are affected so the integrity of our study is not jeopardized.

### Roncesvalles
Within old Toronto, one of the most 'stable' neighbourhoods for food-establishments is Roncesvalles - a historically Polish neighbourhood centred on Roncesvalles Avenue, nestled to the east of High Park. 

While the city-wide average Change Index is 0.2114 and neighbourhood Change Index is 0.2076, food-establishments with the Roncesvalles address are more stable by comparison, averaging a 0.192 Change Index. The table below breaks down each address' Change Index.


```python
roncy = turnover_address[turnover_address['address'].str.contains('RONCESVALLES AVE')]
roncy_ci = roncy['annual_turnover'].sum()/roncy['max_units'].sum()

print("Roncesvalles has {} addresses associated with food-establishments.".format(len(roncy)))
print("The average change_index for Roncesvalles is {:.3f} - lower than the city-wide average by {:.3f} units."\
      .format(roncy_ci, city_ci - roncy_ci))

roncy[['address','change_index']].join(est_by_add.set_index('address'), on='address').groupby('change_index').size()#.sort_values('change_index', ascending = False)
```

    Roncesvalles has 102 addresses associated with food-establishments.
    The average change_index for Roncesvalles is 0.192 - lower than the city-wide average by 0.019 units.
    




    change_index
    0.00000    38
    0.16667    26
    0.25000     2
    0.33333    27
    0.41667     3
    0.50000     3
    0.66667     3
    dtype: int64



### 2.2.3 Turnover by Address <a class ="anchor" id = "turnover_by_a"></a>
The map below shows attributes of each address in the dataset, most notably the names of establishments associated with these addresses. Note that the year beside each establishment name in the **"Esta."** field indicates its first appearance in the dataset. Years after 2012 are likely new openings.


```python
# Create dataframe to feed into map
df = turnover_address.join(est_by_add[['address','Esta.']].set_index('address'), on='address', how = 'left')

df_to_geojson(df,
              filename='markers.geojson',
              properties = ['address','total_turnover','annual_turnover','max_units','change_index','Esta.'],
              lat='LATITUDE', lon='LONGITUDE', precision=6)

# Read private token from local file 
with open('mapbox_token.txt','r') as f:
    token = f.read()

# The leftmost and rightmost bin edges
first_edge, last_edge = df['change_index'].min(), df['change_index'].max()
n_equal_bins = 5
color_breaks = np.linspace(start=first_edge, stop=last_edge, num=n_equal_bins + 1, endpoint=True)

# Generate data breaks and color stops from colorBrewer
color_stops = create_color_stops(np.round(color_breaks,3), colors='YlOrRd')

viz = CircleViz('markers.geojson', access_token=token, 
                radius = 2, center = (-79.342406,43.720250), # DVP/Lawrence
                zoom = 10,
                color_property = 'change_index',
                color_stops=color_stops,
                label_color = '#69F0AE',
                color_default='#69F0AE',
                stroke_color='#a7a7a7',
               stroke_width = 0.1)
viz.show()
```


<iframe id="map", srcdoc="<!DOCTYPE html>
<html>
<head>
<title>mapboxgl-jupyter viz</title>
<meta charset='UTF-8' />
<meta name='viewport'
      content='initial-scale=1,maximum-scale=1,user-scalable=no' />
<script type='text/javascript'
        src='https://api.tiles.mapbox.com/mapbox-gl-js/v0.51.0/mapbox-gl.js'></script>
<link type='text/css'
      href='https://api.tiles.mapbox.com/mapbox-gl-js/v0.51.0/mapbox-gl.css' 
      rel='stylesheet' />

<style type='text/css'>
    body { margin:0; padding:0; }
    .map { position: absolute; top:0; bottom:0; width:100%; }
    .legend {
        background-color: white;
        color: #6e6e6e;
        border-radius: 3px;
        bottom: 10px;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.10);
        font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
        padding: 0;
        position: absolute;
        right: 10px;
        z-index: 1;
        min-width: 100px;
    }
   .legend.horizontal {bottom: 10px; text-align: left;}

    /* legend header */
    .legend .legend-header { border-radius: 3px 3px 0 0; background: white; }
    .legend .legend-title {
        padding: 6px 12px 6px 12px;
        text-shadow: 0 0 2px white;
        text-transform: capitalize;
        text-align: center;
        font-weight: bold !important;
        font-size: 14px;
        font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
        max-width: 160px;
    }
    .legend-title {padding: 6px 12px 6px 12px; text-shadow: 0 0 2px #FFF; text-transform: capitalize; text-align: center; max-width: 160px; font-size: 0.9em; font-weight: bold;}
    .legend.horizontal .legend-title {text-align: left;}

    /* legend items */
    .legend-content {margin: 6px 12px 6px 12px; overflow: hidden; padding: 0; float: left; list-style: none; font-size: 0.8em;}
    .legend.vertical .legend-item {white-space: nowrap;}
    .legend-value {display: inline-block; line-height: 18px; vertical-align: top;}
    .legend.horizontal ul.legend-content li.legend-item .legend-value,
    .legend.horizontal ul.legend-content li.legend-item {display: inline-block; float: left; width: 30px; margin-bottom: 0; text-align: center; height: 30px;}

    /* legend key styles */
    .legend-key {display: inline-block; height: 10px;}
    .legend-key.default, .legend-key.square {border-radius: 0;}
    .legend-key.circle {border-radius: 50%;}
    .legend-key.rounded-square {border-radius: 20%;}
    .legend.vertical .legend-key {width: 10px; margin-right: 5px; margin-left: 1px;}
    .legend.horizontal .legend-key {width: 30px; margin-right: 0; margin-top: 1px; float: left;}
    .legend.horizontal .legend-key.square, .legend.horizontal .legend-key.rounded-square, .legend.horizontal .legend-key.circle {margin-left: 10px; width: 10px;}
    .legend.horizontal .legend-key.line {margin-left: 5px;}
    .legend.horizontal .legend-key.line, .legend.vertical .legend-key.line {border-radius: 10%; width: 20px; height: 3px; margin-bottom: 2px;}

    /* gradient bar alignment */
    .gradient-bar {margin: 6px 12px 6px 12px;}
    .legend.horizontal .gradient-bar {width: 88%; height: 10px;}
    .legend.vertical .gradient-bar {width: 10px; min-height: 50px; position: absolute; bottom: 4px;}

    /* contiguous vertical bars (discrete) */
    .legend.vertical.contig .legend-key {height: 15px; width: 10px;}
    .legend.vertical.contig li.legend-item {height: 15px;}
    .legend.vertical.contig {padding-bottom: 6px;}

</style>

<style>
    .gradient-bar.bordered, .legend-key.bordered { border: solid #a7a7a7 0.1px; }
</style>

</head>
<body>

<div id='map' class='map'></div>

<script type='text/javascript'>

var legendHeader;

function calcColorLegend(myColorStops, title) {

    // create legend
    var legend = document.createElement('div');
    if ('circle' === 'contiguous-bar') {
        legend.className = 'legend vertical contig';
    }
    else {
        legend.className = 'legend vertical';
    }

    legend.id = 'legend';
    document.body.appendChild(legend);

    // add legend header and content elements
    var mytitle = document.createElement('div'),
        legendContent = document.createElement('ul');
    legendHeader = document.createElement('div');
    mytitle.textContent = title;
    mytitle.className = 'legend-title'
    legendHeader.className = 'legend-header'
    legendContent.className = 'legend-content'
    legendHeader.appendChild(mytitle);
    legend.appendChild(legendHeader);
    legend.appendChild(legendContent);

    if (false === true) {
        var gradientText = 'linear-gradient(to right, ';
        var gradient = document.createElement('div');
        gradient.className = 'gradient-bar';
        legend.appendChild(gradient);
    }

    // calculate a legend entries on a Mapbox GL Style Spec property function stops array
    for (p = 0; p < myColorStops.length; p++) {
        if (!!document.getElementById('legend-points-value-' + p)) {
            //update the legend if it already exists
            document.getElementById('legend-points-value-' + p).textContent = myColorStops[p][0];
            document.getElementById('legend-points-id-' + p).style.backgroundColor = myColorStops[p][1];
        }
        else {
            // create the legend if it doesn't yet exist
            var item = document.createElement('li');
            item.className = 'legend-item';

            var key = document.createElement('span');
            key.className = 'legend-key circle';
            key.id = 'legend-points-id-' + p;
            key.style.backgroundColor = myColorStops[p][1];   

            var value = document.createElement('span');
            value.className = 'legend-value';
            value.id = 'legend-points-value-' + p;

            item.appendChild(key);
            item.appendChild(value);
            legendContent.appendChild(item);
            
            data = document.getElementById('legend-points-value-' + p)

            // round number values in legend if precision defined
            if ((typeof(myColorStops[p][0]) == 'number') && (typeof(null) == 'number')) {
                data.textContent = myColorStops[p][0].toFixed(null);
            }
            else {
                data.textContent = myColorStops[p][0];
            }

            // add color stop to gradient list
            if (false === true) {
                if (p < myColorStops.length - 1) {
                    gradientText = gradientText + myColorStops[p][1] + ', ';
                }
                else {
                    gradientText = gradientText + myColorStops[p][1] + ')';
                }
                if ('vertical' === 'vertical') {
                    gradientText = gradientText.replace('to right', 'to bottom');
                }
            }
        }
    }

    if (false === true) {
        // convert to gradient scale appearance
        gradient.style.background = gradientText;

        // hide legend keys generated above
        var keys = document.getElementsByClassName('legend-key');
        for (var i=0; i < keys.length; i++) {
            keys[i].style.visibility = 'hidden';
        }

        if ('vertical' === 'vertical') {
            gradient.style.height = (legendContent.offsetHeight - 6) + 'px';
        }
    }

    // add class for styling bordered legend keys
    if (true) {
        var keys = document.getElementsByClassName('legend-key');
        for (var i=0; i < keys.length; i++) {
            if (keys[i]) {
                keys[i].classList.add('bordered');
            }
        }
        var gradientBars = document.getElementsByClassName('gradient-bar');
        for (var i=0; i < keys.length; i++) {
            if (gradientBars[i]) {
                gradientBars[i].classList.add('bordered');
            }
        }
    }

    // update right-margin for compact Mapbox attribution based on calculated legend width
    var attribMargin = legend.offsetWidth + 15;
    document.getElementsByClassName('mapboxgl-ctrl-attrib')[0].style.marginRight =  attribMargin.toString() + 'px';

}


function generateInterpolateExpression(propertyValue, stops) {
    var expression;
    if (propertyValue == 'zoom') {
        expression = ['interpolate', ['exponential', 1.2], ['zoom']]
    }
    else if (propertyValue == 'heatmap-density') {
        expression = ['interpolate', ['linear'], ['heatmap-density']]
    }
    else {
        expression = ['interpolate', ['linear'], ['get', propertyValue]]
    }

    for (var i=0; i<stops.length; i++) {
        expression.push(stops[i][0], stops[i][1])
    }
    return expression
}


function generateMatchExpression(propertyValue, stops, defaultValue) {
    var expression;
    expression = ['match', ['get', propertyValue]]
    for (var i=0; i<stops.length; i++) {
        expression.push(stops[i][0], stops[i][1])
    }
    expression.push(defaultValue)
    
    return expression
}


function generatePropertyExpression(expressionType, propertyValue, stops, defaultValue) {
    var expression;
    if (expressionType == 'match') {
        expression = generateMatchExpression(propertyValue, stops, defaultValue)
    }
    else {
        expression = generateInterpolateExpression(propertyValue, stops)
    }

    return expression
}

</script>

<!-- main map creation code, extended by mapboxgl/templates/circle.html -->
<script type='text/javascript'>

    mapboxgl.accessToken = 'pk.eyJ1IjoiZmFiaGxjIiwiYSI6ImNqaGt4c2VqMTMydDMzNnAzbG0xNTU4ZTgifQ.noB8ToOGNBZbQZbJH-2F0A';

    var map = new mapboxgl.Map({
        container: 'map',
        attributionControl: false,
        style: 'mapbox://styles/mapbox/light-v9?optimize=true',
        center: [-79.342406, 43.72025],
        zoom: 10,
        pitch: 0,
        bearing: 0,
        scrollZoom: true,
        touchZoom: true,
        doubleClickZoom: true,
        boxZoom: true,
        transformRequest: (url, resourceType) => {
                    if ( url.slice(0,22) == 'https://api.mapbox.com' || 
                        url.slice(0,26) == 'https://a.tiles.mapbox.com' || 
                        url.slice(0,26) == 'https://b.tiles.mapbox.com' ||
                        url.slice(0,26) == 'https://c.tiles.mapbox.com' ||
                        url.slice(0,26) == 'https://d.tiles.mapbox.com') {
                        //Add Mapboxgl-Jupyter Plugin identifier for Mapbox API traffic
                        return {
                           url: [url.slice(0, url.indexOf('?')+1), 'pluginName=PythonMapboxgl&', url.slice(url.indexOf('?')+1)].join('')
                         }
                     }
                     else {
                         //Do not transform URL for non Mapbox GET requests
                         return {url: url}
                     }
                },
    });

    
    
        map.addControl(new mapboxgl.AttributionControl({ compact: true }));

    

    

        map.addControl(new mapboxgl.NavigationControl());

    

    

    
        
            calcColorLegend([[0.0, 'rgb(255,255,178)'], [0.267, 'rgb(254,217,118)'], [0.533, 'rgb(254,178,76)'], [0.8, 'rgb(253,141,60)'], [1.067, 'rgb(240,59,32)'], [1.333, 'rgb(189,0,38)']] , 'change_index');
        
    
        


    

    map.on('style.load', function() {
        
        

        map.addSource('data', {
            'type': 'geojson',
            'data': 'markers.geojson',
            'buffer': 1,
            'maxzoom': 14
        });

        map.addLayer({
            'id': 'label',
            'source': 'data',
            'type': 'symbol',
            'maxzoom': 24,
            'minzoom': 0,
            'layout': {
                
                'text-size' : generateInterpolateExpression('zoom', [[0, 8],[22, 3* 8]] ),
                'text-offset': [0,-1]
            },
            'paint': {
                'text-halo-color': 'white',
                'text-halo-width': generatePropertyExpression('interpolate', 'zoom', [[0,1], [18,5* 1]]),
                'text-color': '#69F0AE'
            }
        }, '' )
        
        map.addLayer({
            'id': 'circle',
            'source': 'data',
            'type': 'circle',
            'maxzoom': 24,
            'minzoom': 0,
            'paint': {
                
                    'circle-color': generatePropertyExpression('interpolate', 'change_index', [[0.0, 'rgb(255,255,178)'], [0.267, 'rgb(254,217,118)'], [0.533, 'rgb(254,178,76)'], [0.8, 'rgb(253,141,60)'], [1.067, 'rgb(240,59,32)'], [1.333, 'rgb(189,0,38)']], '#69F0AE' ),
                    
                'circle-radius' : generatePropertyExpression('interpolate', 'zoom', [[0,2], [22,10 * 2]]),
                'circle-stroke-color': '#a7a7a7',
                'circle-stroke-width': generatePropertyExpression('interpolate', 'zoom', [[0,0.1], [18,5* 0.1]]),
                'circle-opacity' : 1,
                'circle-stroke-opacity' : 1
            }
        }, 'label');
        
        

        // Popups
        
            var popupAction = 'mousemove',
                popupSettings =  {
                    closeButton: false,
                    closeOnClick: false
                };
        

        // Create a popup
        var popup = new mapboxgl.Popup(popupSettings);

        
        
        // Show the popup on mouseover
        map.on(popupAction, 'circle', function(e) {
            
            
                map.getCanvas().style.cursor = 'pointer';
            

            let f = e.features[0];
            let popup_html = '<div><li><b>Location</b>: ' + f.geometry.coordinates[0].toPrecision(6) + 
                ', ' + f.geometry.coordinates[1].toPrecision(6) + '</li>';

            for (key in f.properties) {
                popup_html += '<li><b> ' + key + '</b>: ' + f.properties[key] + ' </li>'
            }

            popup_html += '</div>'
            popup.setLngLat(e.features[0].geometry.coordinates)
                .setHTML(popup_html)
                .addTo(map);
        });

        

        // change cursor to pointer when mouse is over the circle feature layer
        map.on('mouseenter', 'circle', function () {
            map.getCanvas().style.cursor = 'pointer';
        });

        // reset cursor to pointer when mouse leaves the circle feature layer
        map.on('mouseleave', 'circle', function() {
            map.getCanvas().style.cursor = '';
            
                popup.remove();
            
        });
        
        // Fly to on click
        map.on('click', 'circle', function(e) {
            map.easeTo({
                center: e.features[0].geometry.coordinates,
                zoom: map.getZoom() + 1
            });
        });
    });



</script>

</body>
</html>" style="width: 100%; height: 500px;"></iframe>


We can narrow it down to take a look at the 15 addresses with the highest turnover (i.e. change index):


```python
turnover_address_1.sort_values('change_index', ascending = False)[:15].join(est_by_add[['address','Esta.']].set_index('address'), on='address')
```




<div>
<style>
    .dataframe thead tr:only-child th {
        text-align: right;
    }

    .dataframe thead th {
        text-align: left;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>address</th>
      <th>total_turnover</th>
      <th>annual_turnover</th>
      <th>max_units</th>
      <th>change_index</th>
      <th>multi_units</th>
      <th>LATITUDE</th>
      <th>LONGITUDE</th>
      <th>AREA_NAME</th>
      <th>Esta.</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1614</th>
      <td>1192 QUEEN ST E</td>
      <td>8</td>
      <td>1.33333</td>
      <td>1</td>
      <td>1.33333</td>
      <td>0</td>
      <td>43.66312</td>
      <td>-79.33166</td>
      <td>South Riverdale (70)</td>
      <td>Lambretta Pizzeria (2018), P.O. Box 1192 Snack Bar (2015), Rock Lobster (2014), Skwish (2016), The Curzon (2012)</td>
    </tr>
    <tr>
      <th>2938</th>
      <td>3358 DUNDAS ST W</td>
      <td>8</td>
      <td>1.33333</td>
      <td>1</td>
      <td>1.33333</td>
      <td>0</td>
      <td>43.66577</td>
      <td>-79.48203</td>
      <td>Junction Area (90)</td>
      <td>Indilicious Fine Indian Cuisine (2018), Mr Pita Wrap (2012), Q7 Grill And Pastry (2015), Shawarma And Grill (2014), Tri Resto Pizzeria (2017)</td>
    </tr>
    <tr>
      <th>12221</th>
      <td>1808 EGLINTON AVE W</td>
      <td>8</td>
      <td>1.33333</td>
      <td>1</td>
      <td>1.33333</td>
      <td>0</td>
      <td>43.69610</td>
      <td>-79.44950</td>
      <td>Briar Hill-Belgravia (108)</td>
      <td>Doy Doy Doner Point (2012), Doy Doy Restaurant (2014), Hungry Buddies (2016), Shawarma Palace Express (2018), Silk Road Restaurant (2015)</td>
    </tr>
    <tr>
      <th>3262</th>
      <td>425 EDDYSTONE AVE</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.74655</td>
      <td>-79.52479</td>
      <td>Glenfield-Jane Heights (25)</td>
      <td>Ariilon (2018), Kouzies (2015), White Tower Burger (2013), White Tower Restaurant And Food Catering (2014)</td>
    </tr>
    <tr>
      <th>2316</th>
      <td>51 COLBORNE ST</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.64905</td>
      <td>-79.37473</td>
      <td>Church-Yonge Corridor (75)</td>
      <td>An Nain (2014), Chandani Indian Cuisine (2018), Galangal Thai Fusion (2013), Real Mo Mos (2016)</td>
    </tr>
    <tr>
      <th>12081</th>
      <td>760 ST CLAIR AVE W</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.68137</td>
      <td>-79.42848</td>
      <td>Wychwood (94)</td>
      <td>Acquolina Italian Grill (2012), Bywoods (2014), Concession Road (2015), Ji Restaurant (2016)</td>
    </tr>
    <tr>
      <th>6526</th>
      <td>1011 DUFFERIN ST</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.66017</td>
      <td>-79.43523</td>
      <td>Dovercourt-Wallace Emerson-Junction (93)</td>
      <td>Agent Grill (2014), Burger Shawarma (2015), Cindirella Restaurant (2018), Greeko Grill And Cafe (2013)</td>
    </tr>
    <tr>
      <th>10441</th>
      <td>3023 BATHURST ST</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.71814</td>
      <td>-79.42916</td>
      <td>Bedford Park-Nortown (39)</td>
      <td>Au Leaf Cafe Etc. (2017), Dr. Laffa On The Go (2013), Famous Laffa (2015), Urth Cafe Etc (2016)</td>
    </tr>
    <tr>
      <th>3072</th>
      <td>467 DANFORTH AVE</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.67767</td>
      <td>-79.35006</td>
      <td>North Riverdale (68)</td>
      <td>Eggies Breakfast Bistro (2015), Mexico Lindo (2016), Patty And Franks Gourmet Burger And Hotdogs (2013), Starving Artist (2018)</td>
    </tr>
    <tr>
      <th>3651</th>
      <td>839 COLLEGE ST</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.65434</td>
      <td>-79.42293</td>
      <td>Trinity-Bellwoods (81)</td>
      <td>Iberia   Sur (2015), Paul S Churrasco (2013), Shio Japanese Restaurant (2016), Southern Accent Restaurant (2017)</td>
    </tr>
    <tr>
      <th>9546</th>
      <td>4 MANOR RD E</td>
      <td>7</td>
      <td>1.16667</td>
      <td>1</td>
      <td>1.16667</td>
      <td>0</td>
      <td>43.70267</td>
      <td>-79.39717</td>
      <td>Mount Pleasant West (104)</td>
      <td>Coquine Lexpress Cafe (2014), La Bamboche (2013), Saladishes (2016), Zezafoun Syrian Cuisine (2018)</td>
    </tr>
    <tr>
      <th>2093</th>
      <td>106 QUEEN ST E</td>
      <td>6</td>
      <td>1.00000</td>
      <td>1</td>
      <td>1.00000</td>
      <td>0</td>
      <td>43.65379</td>
      <td>-79.37376</td>
      <td>Church-Yonge Corridor (75)</td>
      <td>Ace Jerk Restaurant (2018), Georges Mini Mart (2012), Island Hot And Spicy Restaurant (2016), The Pink Grapefruit (2013)</td>
    </tr>
    <tr>
      <th>1362</th>
      <td>260 MARKHAM RD</td>
      <td>6</td>
      <td>1.00000</td>
      <td>1</td>
      <td>1.00000</td>
      <td>0</td>
      <td>43.74514</td>
      <td>-79.22028</td>
      <td>Scarborough Village (139)</td>
      <td>Caribbean Sweet Spot (2018), Jones Escape (2012), Markham Bar And Grill (2014), Tropics Restaurant/Bar (2016)</td>
    </tr>
    <tr>
      <th>10382</th>
      <td>1753 AVENUE RD</td>
      <td>6</td>
      <td>1.00000</td>
      <td>1</td>
      <td>1.00000</td>
      <td>0</td>
      <td>43.72917</td>
      <td>-79.41815</td>
      <td>Bedford Park-Nortown (39)</td>
      <td>Fusion Work On Avenue (2012), Nawab Express (2016), Nawab Express On Avenue (2018), The Kathi Roll Express (2014)</td>
    </tr>
    <tr>
      <th>1354</th>
      <td>201 MARKHAM RD</td>
      <td>6</td>
      <td>1.00000</td>
      <td>1</td>
      <td>1.00000</td>
      <td>0</td>
      <td>43.74369</td>
      <td>-79.21886</td>
      <td>Scarborough Village (139)</td>
      <td>Ming Hakka House (2012), Peking Express (2014), Samudra Caters (2016), Xing Wang Restaurant (2015)</td>
    </tr>
  </tbody>
</table>
</div>



1192 Queen Street East, 1808 Eglinton Avenue West, and 3358 Dundas Street West are tied for the highest turnover. We can confirm these turnovers with Google Streetview's historical captures. For 1192 Queen Street East, food establishments typically lasted a year before changing hands and a new restaurant was always quick to open in its place. Below is a collection of streetview images of 1192 Queen Street East:
#### 1192 Queen St E
<img src ="1192%20Queen%20St%20E.jpg">
<div style="text-align: center"><i> Source: Google Streetview and BlogTO </i> </div>

## Conclusion <a class ="anchor" id = "conclusion"></a>

From something as simple as food inspection data, we can derive many insights by transforming and reformatting the data. This exploratory analysis has hopefully demonstrated the potential for public data to be used in a way that can spark ideas and confirm hypotheses. 

Other analyses that may be worth pursuing using this data and methodology may be:
- the impact of transit projects (like the Eglinton LRT) on local neighbourhood businesses; 
- correlations between local average housing values over the years to restaurant closures; 
- scraping and joining Yelp data (check if legal) to attribute each restaurant to a price point (or cuisine) and deriving trends from that.

As usual, I'm happy to chat about best practices, different ideas, or learn from your experiences with this type of data. I can be reached on [Twitter](http://www.twitter.com/fabiennechan) and [LinkedIn](https://www.linkedin.com/in/fabiennechan/).

---

### **__Footnotes__ **<a class ="anchor" id = "footnote"></a>
1. Specifically from files dated July 2013, Feb 2015, Jan 201, Sep 2016, Mar 2017, and Dec 2018.
2. Statistics Canada, Census 2016, via City of Toronto Community Council Area Profiles [(Link)](https://www.toronto.ca/wp-content/uploads/2018/05/8f80-City_Planning_2016_Census_Profile_2014_Wards_CCA_Scarborough.pdf)
3. eMarketer (2014): "Immigrants in Canada: Just How Big Is Their Spending Power?" [(Link)](https://www.emarketer.com/Article/Immigrants-Canada-Just-How-Big-Their-Spending-Power/1011356)
