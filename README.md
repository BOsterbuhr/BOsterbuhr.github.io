# Political Donation Analysis
### Which Political Party Recieves More Smaller Donations?


I have always been fascinated with political donations, and what their impact is on politics as a whole. For this project I am going to explore the size of political donations and how they correspond to a political party. All political donations are reported to the Federal Election Commission (FEC) and include details about the donor, so I will use that data to test my hypothesis.

---

Using the FEC's API, I have previously written a program that retrives individual campaign contributions from a specific election year and race. For this project I chose the 2008 Presidential race, and my program retured a csv with ~100,000 row.

## The first step is to import the packages I need.

```python
# Import neccesary packages.
import pandas as pd
import matplotlib.pyplot as plt
from scipy import stats
import seaborn as sns
```

### I uploaded my data to Github to use the permalink to the CSV.
```python
# Import FEC transactions CSV and convert to dataframe.
data = 'https://raw.githubusercontent.com/BOsterbuhr/FEC-Data-Wranglin/main/data/raw_data/1001_of_37294_for_P_in_2008.csv'
fec_df = pd.read_csv(data, index_col=0)
```
|   | contribution_receipt_amount | contributor_state | contributor_zip | party | 
|---|:-----------------------------:|:-------------------:|:-----------------:|:-------:| 
| 0 | 500.0                       | NC                | 27006           | REP   | 
| 1 | 10.0                        | CA                | 92801           | REP   | 
| 2 | 10.0                        | CA                | 92801           | REP   | 
| 3 | 16.0                        | NJ                | 8889            | REP   | 
| 4 | 20.0                        | CA                | 93420           | REP   | 



### The data contains contributions from outside the continental United States, which makes mapping less visually appealing. 
To deal with that I removed any contribution outside the continental U.S.
```python
# Remove any transaction from outside the continental U.S. for better mapping later.
map_fec_df = fec_df.copy()
keep = ['AL', 'AR', 'AZ', 'CA', 'CO',
        'CT', 'DC', 'DE', 'FL', 'GA',
        'IA', 'ID', 'IL', 'IN', 'KS',
        'KY', 'LA', 'MA', 'MD', 'ME',
        'MI', 'MN', 'MO', 'MS', 'MT', 
        'NC', 'ND', 'NE', 'NH', 'NJ', 
        'NM', 'NV', 'NY', 'OH', 'OK', 
        'OR', 'PA', 'RI', 'SC', 'SD', 
        'TN', 'TX', 'UT', 'VA', 'VT', 
        'WA', 'WI', 'WV', 'WY']
index_names = map_fec_df[ ~map_fec_df['contributor_state'].isin(keep) ].index
map_fec_df.drop(index_names, inplace=True)

# Check the length of data.
print(len(map_fec_df))
print(len(fec_df))
```
>98625  
100100

### To accurately plot the contributions I want to convert my zip codes to latitude (lat) and longitude (lng) coordinates. To do so, I will import another dataset containing the lat and lng for every zip code.

```python
# Import CSV to get latitude and longitude for each row's zip code, then assign to dataframe. 
lat_long = 'https://gist.githubusercontent.com/erichurst/7882666/raw/5bdc46db47d9515269ab12ed6fb2850377fd869e/US%2520Zip%2520Codes%2520from%25202013%2520Government%2520Data'
lat_long_df = pd.read_csv(lat_long)
```
| ZIP   | LAT       | LNG         | 
|:-------:|:-----------:|:-------------:| 
| 00601 | 18.180555 |  -66.749961 | 
| 00602 | 18.361945 |  -67.175597 | 
| 00603 | 18.455183 |  -67.119887 | 
| 00606 | 18.158345 |  -66.932911 | 
| 00610 | 18.295366 |  -67.125135 | 

### Now to merge the two dataframes into one containing `LAT` and `LNG` for every row in `map_fec_df`. 
```python
# Merge map_fec_df and lat_long_df where the zip codes match, saving the rows from  map_fec_df and getting rid of any unused rows from lat_long_df.
merged_df = pd.merge(map_fec_df, lat_long_df, how='left', left_on='contributor_zip', right_on='ZIP')
merged_df = merged_df.drop(['ZIP','contributor_zip'], axis=1)
```

|      | contribution_receipt_amount | contributor_state | party | LAT       | LNG         | 
|:---------:|:-----------------------------:|:-------------------:|:-------:|:-----------:|:-------------:| 
| 0       | 500.0                       | NC                | REP   | 35.939004 | -80.440264  | 
| 1       | 10.0                        | CA                | REP   | 33.844983 | -117.952151 | 
| 2       | 10.0                        | CA                | REP   | 33.844983 | -117.952151 | 
| 3       | 16.0                        | NJ                | REP   | 40.609021 | -74.754216  | 
| 4       | 20.0                        | CA                | REP   | 35.176043 | -120.476694 | 

## Data Cleaning Time!

```python
# Function to encode party column.
def encode_party(party):
    if party == 'REP':
      return 2
    elif party == "DEM":
      return 0
    else:
      return 1

# New column for the encoded party values.
merged_df['party_code'] = merged_df['party'].apply(encode_party)
print(merged_df.head())
```
|     | contribution_receipt_amount  | contributor_state | party | LAT       | LNG            | party_code | 
|:------:|:------------------------------:|:-------------------:|:-------:|:-----------:|:----------------:|:------------:| 
| 0    | 500.0                        | NC                | REP   | 35.939004 | -80.440264     | 2          | 
| 1    | 10.0                         | CA                | REP   | 33.844983 | -117.952151    | 2          | 
| 2    | 10.0                         | CA                | REP   | 33.844983 | -117.952151    | 2          | 
| 3    | 16.0                         | NJ                | REP   | 40.609021 | -74.754216     | 2          | 
| 4    | 20.0                         | CA                | REP   | 35.176043 | -120.476694    | 2          | 
```python
# More df cleaning, this time dropping any rows with negative contribution_receipt_amount
index_names = merged_df[ merged_df['contribution_receipt_amount'] < 0 ].index
merged_df.drop(index_names, inplace=True)
print(len(merged_df))
```
> 93228
```python
# The evem after specifing continental U.S. states earlier there are still outliers. So I need to remove them to make the scatter plot look nice.
index_names = merged_df[ merged_df['LAT'] < 24 ].index
merged_df.drop(index_names, inplace=True)
print(len(merged_df))
```
> 93225
```python
# Separte versions of merged_df into Democrat and Republican dataframes.
rep_df = merged_df[merged_df['party_code'] == 2]
print('Republican length: 'len(rep_df))
dem_df = merged_df[merged_df['party_code'] == 0]
print('Democrat length: 'len(dem_df))
```
> Republican length: 22605  
> Democrat length: 69688

```python
rep_df.plot(kind='scatter', x='LNG', y='LAT', s=rep_df['contribution_receipt_amount']/100, label='Rep Donation Amount', c='#FF0000', alpha=0.4, figsize=(10,7))
dem_df.plot(kind='scatter', x='LNG', y='LAT', s=dem_df['contribution_receipt_amount']/100, label='Dem Donation Amount', c='#0000FF', alpha=0.4, figsize=(10,7))
plt.legend()
plt.show()
```

### `Rep_df`:
![alt text](https://github.com/BOsterbuhr/BOsterbuhr.github.io/blob/527408e4fe2db19d7dad8703331752a283f0c828/rep_df.png?raw=true "Republican Donations")

### `Dem_df`:
![alt text](https://github.com/BOsterbuhr/BOsterbuhr.github.io/blob/527408e4fe2db19d7dad8703331752a283f0c828/dem_df.png?raw=true "Democrat Donations")

```python
def donation_size(donation):
  if donation > 100:
    return 1
  else:
    return 0
fec_df['Over_100'] = fec_df['contribution_receipt_amount'].apply(donation_size)
fec_df['party_code'] = fec_df['party'].apply(encode_party)
fec_df.head()
```
|  | contribution_receipt_amount | contributor_state | contributor_zip | party | Over_100 | 
|:---:|:-----------------------------:|:-------------------:|:-----------------:|:-------:|:----------:| 
| 0 | 500.0                       | NC                | 27006           | REP   | 1        | 
| 1 | 10.0                        | CA                | 92801           | REP   | 0        | 
| 2 | 10.0                        | CA                | 92801           | REP   | 0        | 
| 3 | 16.0                        | NJ                | 8889            | REP   | 0        | 
| 4 | 20.0                        | CA                | 93420           | REP   | 0        |

## Finally time to form a hypothesis and test it.
I want to know whether there is a statistically significant difference in the proportion of donations over $100 between Democrats and Republicans. So I will do a two sided t-test on the Over_100 column.

Our hypothesis will be:

𝐻0:𝜇1=𝜇2 vs.  𝐻𝑎:𝜇1≠𝜇2

Where 𝜇1 is the mean of Democrat `Over_100` column and 𝜇2 is the mean of Republican `Over_100` column.
```python
test_rep_df = fec_df[fec_df['party'] == 'REP']
test_dem_df = fec_df[fec_df['party'] == 'DEM']
two_party_tval, two_party_pval = stats.ttest_ind(test_dem_df['Over_100'], test_rep_df['Over_100'])
print([two_party_tval, two_party_pval])
```
> [10.069977672916938, 7.70025338804504e-24]

## From these results we can safely reject the null hypothesis and conclude that the difference in the population proportions are statistically significant.
# So awesome but what does that actually mean? What party actually has a larger proportion of donations over 100? To answer that lets do some crosstabs!

```python
dem_donation_crosstab = pd.crosstab(test_dem_df['party'], test_dem_df['Over_100'], normalize=True) * 100
rep_donation_crosstab = pd.crosstab(test_rep_df['party'], test_rep_df['Over_100'], normalize=True) * 100
```
| Over_100               | No    | Yes   | 
|------------------------|:-------:|:-------:| 
| party                  |       |       | 
| DEM                    | 78.073085 | 21.926915 | 
| REP                    | 80.990077 | 19.009923  | 

### As you can see when we normalize the `'Over_100'` column we can confirm that the `'Over_100'` are not the same. Normalizing the results gives a clearer picture than what `value_counts()` would provide. As shown below.

```python
two_party_donation_crosstab = rep_donation_crosstab.merge(dem_donation_crosstab, how='outer')
this_is_getting_ridiculous = two_party_donation_crosstab.rename(columns={0:'Under $100', 1:'Over $100'}, index={0:'Republicans', 1:'Democrats'})
this_is_getting_ridiculous.plot.barh(grid=True)
plt.show()
```
![alt text]('Crosstab')

# These results will need to be tested again with more samples to draw any conclusions about political donations overall. An import thing to consider is the data contains the last ~100,000 donations of the 2008 presidential election when President Obama was first elected. 

## Now for fun I took too long to made this plot:
```python
from matplotlib.colors import LinearSegmentedColormap
colors = [(0, 0, 1), (0, 1, 0), (1, 0, 0)]  # R -> G -> B
n_bin = 10
cmap_name = 'my_list'
cm = LinearSegmentedColormap.from_list(cmap_name, colors, N=n_bin)
fig = plt.figure()
fig.set_size_inches(20,16)
max_lat = merged_df['LAT'].max()
min_lat = merged_df['LAT'].min()
max_lng = merged_df['LNG'].max()
min_lng = merged_df['LNG'].min()
im = plt.imread('https://github.com/BOsterbuhr/LambdaSchool/blob/main/Projects/Media/USA_location_map_-_counties_-_no_water.png?raw=true')
implot = plt.imshow(im, extent=[min_lng - .8, max_lng + .58, min_lat - .15, max_lat + .75])
plt.scatter( x=merged_df['LNG'], y=merged_df['LAT'], s=merged_df['contribution_receipt_amount']/100,
            label='Donation Amount', c=merged_df["party_code"], cmap=cm, alpha=.25)
plt.legend()
plt.title('2008 Presidential Campaign Donations')
plt.ylabel('Latitude')
plt.xlabel('Longitude')
plt.show()
```
![alt text](https://github.com/BOsterbuhr/BOsterbuhr.github.io/blob/527408e4fe2db19d7dad8703331752a283f0c828/2008PresidentialCampaignDonations.jpeg?raw=true 'Full US')
