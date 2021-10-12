```python
import pandas as pd
import numpy as np
import geopandas as gp
import geojson
from matplotlib import pyplot as plt
```

# Intro
The 50 states in U.S. have different areas, ranging from 4,001 km2 (RI) to 1,723,337 km2 (AK). This could sometimes cause huge challenges in visualization since smaller states would be hard to represent. 

Using a matrix map could help solve this problem. By abstract the US map into same-sized shapes while preserving the relative geographical positions, we could convey the information more precisely without bias when the area doesn't matter.

![US Matrix Map Sample](us_matrix_map_sample.png)

## Prepare Map Matrix

First we can use a spreadsheet to build up a matrix to abstract the real map. It looks something like below.


```python
# Read in the data I prepared
us_map = pd.read_csv('desktop/OTD Goal/us_matrix_map.csv',header=None)
us_map
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>0</th>
      <th>1</th>
      <th>2</th>
      <th>3</th>
      <th>4</th>
      <th>5</th>
      <th>6</th>
      <th>7</th>
      <th>8</th>
      <th>9</th>
      <th>10</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AK</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>ME</td>
    </tr>
    <tr>
      <th>1</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>WI</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>VT</td>
      <td>NH</td>
    </tr>
    <tr>
      <th>2</th>
      <td>WA</td>
      <td>ID</td>
      <td>MT</td>
      <td>ND</td>
      <td>MN</td>
      <td>IL</td>
      <td>MI</td>
      <td>NaN</td>
      <td>NY</td>
      <td>MA</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>3</th>
      <td>OR</td>
      <td>NV</td>
      <td>WY</td>
      <td>SD</td>
      <td>IA</td>
      <td>IN</td>
      <td>OH</td>
      <td>PA</td>
      <td>NJ</td>
      <td>CT</td>
      <td>RI</td>
    </tr>
    <tr>
      <th>4</th>
      <td>CA</td>
      <td>UT</td>
      <td>CO</td>
      <td>NE</td>
      <td>MO</td>
      <td>KY</td>
      <td>WV</td>
      <td>VA</td>
      <td>MD</td>
      <td>DE</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>5</th>
      <td>NaN</td>
      <td>AZ</td>
      <td>NM</td>
      <td>KS</td>
      <td>AR</td>
      <td>TN</td>
      <td>NC</td>
      <td>SC</td>
      <td>DC</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>6</th>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>OK</td>
      <td>LA</td>
      <td>MS</td>
      <td>AL</td>
      <td>GA</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>7</th>
      <td>HI</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>TX</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>NaN</td>
      <td>FL</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Flatten the matrix
us_map = us_map.stack().reset_index()
us_map.columns = ['row','column','state']
us_map = us_map[us_map['state']!='']
us_map['row'] = us_map['row'].max() - us_map['row'] 
us_map.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>row</th>
      <th>column</th>
      <th>state</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>7</td>
      <td>0</td>
      <td>AK</td>
    </tr>
    <tr>
      <th>1</th>
      <td>7</td>
      <td>10</td>
      <td>ME</td>
    </tr>
    <tr>
      <th>2</th>
      <td>6</td>
      <td>5</td>
      <td>WI</td>
    </tr>
    <tr>
      <th>3</th>
      <td>6</td>
      <td>9</td>
      <td>VT</td>
    </tr>
    <tr>
      <th>4</th>
      <td>6</td>
      <td>10</td>
      <td>NH</td>
    </tr>
  </tbody>
</table>
</div>



## Generate Geojson File
Now we use the geojson module to translate our prepared data into a geojson file.


```python
# Here I make a geojson file that can hold 1 overall value and 3 subtotal value for each state
features = []
value_holders = {'overall': {'start':[0,0], 'width':3, 'height':2},
          'subtotal1': {'start':[0,2], 'width':1, 'height':1},
          'subtotal2': {'start':[1,2], 'width':1, 'height':1},
          'subtotal3': {'start':[2,2], 'width':1, 'height':1}}
step = 4

for state in us_map.iterrows():
    state_name = state[1]['state']
    main_start_point_x = state[1]['column']
    main_start_point_y = state[1]['row']
    for value_holder in value_holders:
        aug = value_holders[value_holder]
        start_point_x = main_start_point_x * step + aug['start'][0] 
        start_point_y = main_start_point_y * step + aug['start'][1]
        feature = geojson.Feature(state_name, geojson.Polygon([[(start_point_x, start_point_y),
                                                                (start_point_x+aug['width'],start_point_y),
                                                                (start_point_x+aug['width'],start_point_y+aug['height']),
                                                                (start_point_x,start_point_y+aug['height']),
                                                                (start_point_x,start_point_y)]]), 
                                  properties={'type':value_holder})
        features.append(feature)
features = geojson.FeatureCollection(features)

with open('us_matrix_map.geojson','w') as w:
    w.write(geojson.dumps(features))
```

## Using the Map
Now that we have the geojson file, we can use it to create matrix maps using multiple modules or softwares. Here I'll use Geopandas to make a sample map. The flexibility of Geopandas allow us to further edit the map's appearance.


```python
# Read in the geojson file with Geopandas
us_map = gp.read_file('us_matrix_map.geojson')
us_map.head()
```




<div>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>id</th>
      <th>type</th>
      <th>geometry</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AK</td>
      <td>overall</td>
      <td>POLYGON ((0.00000 28.00000, 3.00000 28.00000, ...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AK</td>
      <td>subtotal1</td>
      <td>POLYGON ((0.00000 30.00000, 1.00000 30.00000, ...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>AK</td>
      <td>subtotal2</td>
      <td>POLYGON ((1.00000 30.00000, 2.00000 30.00000, ...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AK</td>
      <td>subtotal3</td>
      <td>POLYGON ((2.00000 30.00000, 3.00000 30.00000, ...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>ME</td>
      <td>overall</td>
      <td>POLYGON ((40.00000 28.00000, 43.00000 28.00000...</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Generate some random data
us_map['data'] = np.random.randn(len(us_map))
us_map.head()
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
      <th>id</th>
      <th>type</th>
      <th>geometry</th>
      <th>data</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>AK</td>
      <td>overall</td>
      <td>POLYGON ((0.00000 28.00000, 3.00000 28.00000, ...</td>
      <td>-0.780820</td>
    </tr>
    <tr>
      <th>1</th>
      <td>AK</td>
      <td>subtotal1</td>
      <td>POLYGON ((0.00000 30.00000, 1.00000 30.00000, ...</td>
      <td>-1.072805</td>
    </tr>
    <tr>
      <th>2</th>
      <td>AK</td>
      <td>subtotal2</td>
      <td>POLYGON ((1.00000 30.00000, 2.00000 30.00000, ...</td>
      <td>1.314921</td>
    </tr>
    <tr>
      <th>3</th>
      <td>AK</td>
      <td>subtotal3</td>
      <td>POLYGON ((2.00000 30.00000, 3.00000 30.00000, ...</td>
      <td>-1.027133</td>
    </tr>
    <tr>
      <th>4</th>
      <td>ME</td>
      <td>overall</td>
      <td>POLYGON ((40.00000 28.00000, 43.00000 28.00000...</td>
      <td>0.299534</td>
    </tr>
  </tbody>
</table>
</div>




```python
# Now lets make some maps!
fig = plt.figure(dpi=200,figsize=(10,6))
us_map.plot('data', ax=plt.gca(), cmap='RdYlBu_r', linewidth=0.1, edgecolor='grey', vmin=-5, vmax=5)
plt.axis(False)

# Add some text
us_map['center'] = us_map['geometry'].centroid.apply(lambda c: (c.x, c.y))
for row in us_map.iterrows():
    size = 5
    text = '{:.1f}'.format(row[1]['data'])
    if row[1]['type'] == 'overall':
        text = row[1]['id']+'\n'+text
        size = 8
    plt.annotate(text, row[1]['center'], horizontalalignment='center', verticalalignment='center', size=size)
```

    <ipython-input-7-06b8f72af8ba>:7: UserWarning: Geometry is in a geographic CRS. Results from 'centroid' are likely incorrect. Use 'GeoSeries.to_crs()' to re-project geometries to a projected CRS before this operation.
    
      us_map['center'] = us_map['geometry'].centroid.apply(lambda c: (c.x, c.y))



    
![png](output_11_1.png)
    



```python
## We can also use Geopandas to adjust the shapes like changing them into circles!
us_map['geometry'] = us_map['geometry'].centroid.buffer(0.5)
us_map.loc[us_map['type']=='overall', 'geometry'] = us_map['geometry'].buffer(0.5)

fig = plt.figure(dpi=200,figsize=(10,6))
us_map.plot('data', ax=plt.gca(), cmap='RdYlBu_r', linewidth=0.1, edgecolor='grey', vmin=-5, vmax=5)
plt.axis(False)

# Add some text
us_map['center'] = us_map['geometry'].centroid.apply(lambda c: (c.x, c.y))
for row in us_map.iterrows():
    size = 5
    text = '{:.1f}'.format(row[1]['data'])
    if row[1]['type'] == 'overall':
        text = row[1]['id']+'\n'+text
        size = 8
    plt.annotate(text, row[1]['center'], horizontalalignment='center', verticalalignment='center', size=size)
```

    <ipython-input-8-f1e368862819>:2: UserWarning: Geometry is in a geographic CRS. Results from 'centroid' are likely incorrect. Use 'GeoSeries.to_crs()' to re-project geometries to a projected CRS before this operation.
    
      us_map['geometry'] = us_map['geometry'].centroid.buffer(0.5)
    <ipython-input-8-f1e368862819>:2: UserWarning: Geometry is in a geographic CRS. Results from 'buffer' are likely incorrect. Use 'GeoSeries.to_crs()' to re-project geometries to a projected CRS before this operation.
    
      us_map['geometry'] = us_map['geometry'].centroid.buffer(0.5)
    <ipython-input-8-f1e368862819>:3: UserWarning: Geometry is in a geographic CRS. Results from 'buffer' are likely incorrect. Use 'GeoSeries.to_crs()' to re-project geometries to a projected CRS before this operation.
    
      us_map.loc[us_map['type']=='overall', 'geometry'] = us_map['geometry'].buffer(0.5)
    <ipython-input-8-f1e368862819>:10: UserWarning: Geometry is in a geographic CRS. Results from 'centroid' are likely incorrect. Use 'GeoSeries.to_crs()' to re-project geometries to a projected CRS before this operation.
    
      us_map['center'] = us_map['geometry'].centroid.apply(lambda c: (c.x, c.y))



    
![png](output_12_1.png)
    



```python

```
