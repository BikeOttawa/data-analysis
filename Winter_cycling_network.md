---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.3'
      jupytext_version: 1.11.3
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

# Winter Cycling Network missing links analysis

Goal: identify prime areas for expansion of the Winter Cycling Network, areas that would provide the most return on investment for increasing ridership and usefulness of the network.

Analysis plan:
- [x] **Fall to winter comparison**: calculate the relative change in ridership on between September & October 2019 and January & Februrary 2020 (normalized to the total per month) and plot on a map over the Winter Cycling Network
    - Why: Fall has a higher fraction of utility cycling trips than summer; people who cycle in the fall but stop in the winter would be prime candidates for using an expanded Winter Cycling Network.

To do:
- [x] Load September 2019 and January 2020 Strava data
- [x] Load OSM tags for the Winter Cycling Network
- [x] Calculate relative change in ridership between September and January, normalized to total monthly counts (or total people?)
    - [ ] Possibly split by weekday / weekend, or use just commute trips on Strava?
    - [ ] Split by men and women
- [x] Plot Winter Cycling Network and relative change on the same map

Strava user totals:

| --- | Total activities | Total people |
| ---- | ----------- | -------------| 
| September 2019 | 46,283 | 6,079 | 
| October 2019 | 30,176 | 3,826 | 
| January 2020 | 6,033 | 663  |
| February 2020 | 5,268 | 758  |

Right off the top we can notice that fewer people are making more trips in January: the ratio of trips to people is about 9:1, whereas in September it's about 7.5:1. Winter cyclists are more likely to be quite dedicated cyclists?

```python
# load packages
import pandas as pd
import numpy as np
import matplotlib
from mpl_toolkits.axes_grid1 import make_axes_locatable
from matplotlib import pyplot as plt
import seaborn as sns
import geopandas as gpd # for shapefile
import contextily as ctx # for background map
```

```python
# load data
# data can be accessed on the Strava Metro downloads page for the Bike Ottawa organization
Sept_2019 = pd.read_csv("data/Strava/all_edges_daily_2019-09-01-2019-09-30_ride" 
                        + "/1b5bb8a5a94591250339fa667a1bd40b690291651b529a30a0e0bbb732ea6a0f-1634409075566.csv")
Oct_2019 = pd.read_csv("data/Strava/all_edges_daily_2019-10-01-2019-10-31_ride"
                       + "/c4f3750259f4329081fc5e53cc782d5b2fd0b305cc4c7f2ba1e3cde95b8c6d58-1637607788760.csv")
Jan_2020 = pd.read_csv("data/Strava/all_edges_daily_2020-01-01-2020-01-31_ride"
                       + "/a92b60d941fcc0a525ada734d70ec2fb2aabf785ba28d1eaa9ad7293bab8559b-1634514902350.csv")
Feb_2020 = pd.read_csv("data/Strava/all_edges_daily_2020-02-01-2020-02-29_ride"
                       +"/18bf99a933c28df850a2268a62702fa98b5444359041c4e1f065771495110bc0-1637634385929.csv")
```

```python
# shapefile
Sept_2019_shape = gpd.read_file("data/Strava/all_edges_daily_2019-09-01-2019-09-30_ride"
                           +"/1b5bb8a5a94591250339fa667a1bd40b690291651b529a30a0e0bbb732ea6a0f-1634409075566.shp")
Oct_2019_shape = gpd.read_file("data/Strava/all_edges_daily_2019-10-01-2019-10-31_ride"
                       + "/c4f3750259f4329081fc5e53cc782d5b2fd0b305cc4c7f2ba1e3cde95b8c6d58-1637607788760.shp")
Jan_2020_shape = gpd.read_file("data/Strava/all_edges_daily_2020-01-01-2020-01-31_ride"
                           "/a92b60d941fcc0a525ada734d70ec2fb2aabf785ba28d1eaa9ad7293bab8559b-1634514902350.shp")
Feb_2020_shape = gpd.read_file("data/Strava/all_edges_daily_2020-02-01-2020-02-29_ride"
                       +"/18bf99a933c28df850a2268a62702fa98b5444359041c4e1f065771495110bc0-1637634385929.shp")
```

```python
# rename edgeUID for join
Sept_2019_shape = Sept_2019_shape.rename(columns={"edgeUID": "edge_uid"})
Oct_2019_shape = Oct_2019_shape.rename(columns={"edgeUID": "edge_uid"})
Jan_2020_shape = Jan_2020_shape.rename(columns={"edgeUID": "edge_uid"})
Feb_2020_shape = Feb_2020_shape.rename(columns={"edgeUID": "edge_uid"})
```

```python
# load Winter Cycling Network OSM features
wcn = gpd.read_file("data/winter-map.geojson")
```

```python
# ahhhhh, so easy, love geopandas
wcn.plot()
```

```python
# total Strava users for each month
user_totals = [6079, 3826, 663, 758] # September, October, January, February
```

```python
# combine counts in each direction on all segments
for i, df in enumerate([Sept_2019, Oct_2019, Jan_2020, Feb_2020]):
    df['total_trips'] = df['forward_trip_count'] + df['reverse_trip_count']
    df['total_female_people'] = df['forward_female_people_count'] + df['reverse_female_people_count']
    df['total_male_people'] = df['forward_male_people_count'] + df['reverse_male_people_count']
    df['normalized_trips'] = df['total_trips'] / user_totals[i]
    df['female_normalized'] = df['total_female_people'] / user_totals[i]
    df['male_normalized'] = df['total_male_people'] / user_totals[i]

```

```python
# stack dataframes together
Strava = pd.concat([Sept_2019, Oct_2019, Jan_2020, Feb_2020])
```

```python
# drop columns with data we're not using
Strava = Strava.drop(columns = ['forward_people_count', 'reverse_people_count',
       'forward_commute_trip_count', 'reverse_commute_trip_count',
       'forward_leisure_trip_count', 'reverse_leisure_trip_count',
       'forward_morning_trip_count', 'reverse_morning_trip_count',
       'forward_evening_trip_count', 'reverse_evening_trip_count',
       'forward_male_people_count', 'reverse_male_people_count',
       'forward_female_people_count', 'reverse_female_people_count',
       'forward_unspecified_people_count', 'reverse_unspecified_people_count',
       'forward_13_19_people_count', 'reverse_13_19_people_count',
       'forward_20_34_people_count', 'reverse_20_34_people_count',
       'forward_35_54_people_count', 'reverse_35_54_people_count',
       'forward_55_64_people_count', 'reverse_55_64_people_count',
       'forward_65_plus_people_count', 'reverse_65_plus_people_count',
       'forward_average_speed', 'reverse_average_speed', 
                               'activity_type'] )
```

```python
# make a 'year' column
Strava['year'] = Strava['date'].str.split('-', expand = True)[0]
Strava['year'] = Strava['year'].astype('int')
```

```python
months_grouped = Strava.groupby(['edge_uid', 'year'])[['total_trips', 'normalized_trips']].sum().reset_index()
```

```python
# create a single shapefile by merging all the shapefiles (should be lots of overlap, possibly complete overlap)
shape = Sept_2019_shape.merge(Oct_2019_shape, on = ['edge_uid', 'osmId', 'geometry'], how = 'outer')
shape = shape.merge(Jan_2020_shape, on = ['edge_uid', 'osmId', 'geometry'], how = 'outer')
shape = shape.merge(Feb_2020_shape, on = ['edge_uid', 'osmId', 'geometry'], how = 'outer')
```

```python
# create shape subset in the region of the WCN
east_lim = -75.64
west_lim = -75.76
north_lim = 45.45
south_lim = 45.37
shape_subset = shape.cx[west_lim:east_lim, south_lim: north_lim]
```

```python
# merge the shape data with the Strava counts using the 'edge_uid' column
Strava_osm = shape_subset.merge(months_grouped, on = 'edge_uid', how = 'inner')
```

```python
# take a look at the data
Strava_osm.head()
```

```python
# pivot to wide format to calculate percentage change for each year
Strava_pivot = Strava_osm.pivot(index = (['year']), columns = (['edge_uid']), 
                                values = 'normalized_trips').reset_index()
```

```python
# fill missing values with 0: if an edge is not there, it means 0 counts
# pivot to wide format to calculate percentage change for each year
Strava_pivot = Strava_osm.pivot(index = (['year']), columns = (['edge_uid']), 
                                values = 'normalized_trips').reset_index().fillna(0)

edge_pct_change = Strava_pivot.pct_change()*100
```

```python
edge_pct_change
```

```python
# rename year columns
edge_pct_change['year'] = [2019, 2020]
```

```python
# melt back to long version
pct_change_2019_2020 = edge_pct_change.iloc[1:].melt(id_vars = ['year'])
```

```python
# rename columns
pct_change_2019_2020 = pct_change_2019_2020.rename(columns = {"value": "pct_change"})
```

```python
# merge with the shape data again
# this will remove any edges that aren't in the 2020 data, which I don't exactly want...
# drop duplicates first

Strava_fall_winter = Strava_osm[['edge_uid', 'osmId', 
                    'geometry']].drop_duplicates().merge(pct_change_2019_2020, 
                                                         on = 'edge_uid', how = 'inner')
```

```python
# change to web mercator for adding basemap
Strava_web_mercator = Strava_fall_winter.to_crs(epsg=3857)
wcn_web_mercator = wcn.to_crs(epsg=3857)
```

```python
fig, ax = plt.subplots(figsize = (10,10))
vmin = -100
vmax = 100

ax.set_aspect('equal')

divider = make_axes_locatable(ax)

cax = divider.append_axes("right", size="5%", pad=0.1)


wcn_web_mercator.plot(ax = ax, linewidth = 4, alpha = 0.5, color = 'green')

Strava_web_mercator.plot(column = "pct_change", ax = ax, cax = cax, legend = True, cmap = 'plasma', 
                       vmin=vmin, vmax=vmax, linewidth = 1)
ctx.add_basemap(ax, alpha = 0.5)

plt.tight_layout()

plt.savefig("wnc_Fall2019_Winter2020.pdf")
```

```python
# change to web mercator for adding basemap
Strava_web_mercator = Strava_osm.to_crs(epsg=3857)
Strava_2019_web_mercator = Strava_web_mercator[Strava_web_mercator['year'] == 2019]
Strava_2020_web_mercator = Strava_web_mercator[Strava_web_mercator['year'] == 2020]
```

```python
import matplotlib.colors as colors
```

```python
fig, ax = plt.subplots(figsize = (10,10))

data = Strava_2019_web_mercator['total_trips']

vmin =  np.min(data)
vmax =  np.max(data)

ax.set_aspect('equal')

divider = make_axes_locatable(ax)

cax = divider.append_axes("right", size="5%", pad=0.1)


wcn_web_mercator.plot(ax = ax, linewidth = 4, alpha = 0.5, color = 'green')

Strava_2019_web_mercator.plot(column = "total_trips", ax = ax, cax = cax, legend = True, cmap = 'plasma', 
                            norm = colors.LogNorm(vmin=vmin, vmax=vmax),)
ctx.add_basemap(ax, alpha = 0.5)

plt.tight_layout()

plt.savefig("Fall2019.pdf")
```

```python
fig, ax = plt.subplots(figsize = (10,10))
data = Strava_2020_web_mercator['total_trips']

vmin =  np.min(data)
vmax =  np.max(data)


ax.set_aspect('equal')

divider = make_axes_locatable(ax)

cax = divider.append_axes("right", size="5%", pad=0.1)


wcn_web_mercator.plot(ax = ax, linewidth = 4, alpha = 0.5, color = 'green')

Strava_2020_web_mercator.plot(column = "total_trips", ax = ax, cax = cax, legend = True, cmap = 'plasma', 
                            norm = colors.LogNorm(vmin=vmin, vmax=vmax),)
ctx.add_basemap(ax, alpha = 0.5)

plt.tight_layout()

plt.savefig("Winter2020.pdf")
```

```python
fig, ax = plt.subplots(figsize = (10,10))
wcn_web_mercator.plot(ax = ax, linewidth = 3, alpha = 0.5, color = 'green')
ctx.add_basemap(ax, alpha = 0.8)
plt.savefig("wcn.pdf")
plt.savefig("wcn.png", dpi = 200)
```

```python

```
