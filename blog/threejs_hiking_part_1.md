<!-- # Visualizing Strava acitivities with ThreeJS -->

Buon giorno! 
Summer is upon us and with it comes extra motivation not to waste our precious time sitting in front of a screen and do something a little bit more meaningful instead.
Many venture to the south to find themselves chilling with a cool drink in their hands on a beach, while others go there for slightly less reasonable activities, e.g. hiking. I happen to be one of the latter because apparently I prefer burning at ~2K altitude for most of the day over doing some sort of light physical activity at a reasonable hour and relaxing later at a waterfront. Long story short, last month I visited Italy (which is, by the way, stupid beautiful no matter what you are up to) and recorded all my activities on Strava.
Let's go waste some time in front of a screen again.

## Data preparation
Alright, so I do have all the track data recorded and I do have a vision. I know jacks#it about geographic systems though (pardon my language), and arguably very little about the 3D space, so let's start with something I'm actually somewhat familiar with and that is data extraction and manipulation. If your are in for `threejs` stuff you can skip this article altogether and go to the next one, since in this part I will be using `python` only. 

For those that do not know, [Strava](https://strava.com/) is a activity tracking app with a very well documented and user-friendly [API](https://developers.strava.com/docs/reference/). Let's look at some endpoints. `athlete/activities` will get us a list of all activities while `activities/{id}` will get a specific one with some additional information. We will have to authenticate ourselves first, and for that we will have to provide `client_id`, `client_secret` and `refresh_token` all of which can be found in the [settings](https://www.strava.com/settings/api) of the app. Next, we can get our `access_token` with the following function using the `requests` library.

```python
def get_access_token(client_id, client_secret, refresh_token): 
   url = 'https://www.strava.com/oauth/token' 
   payload = {
       'client_id': client_id, 
       'client_secret': client_secret, 
       'refresh_token': refresh_token, 
       'grant_type': 'refresh_token', 
       'f': 'json', 
   } 
   response = requests.post(url, data=payload, verify=False) 
   access_token = response.json().get('access_token')
   return access_token

```
Fairly simple now that I look at it, but I have to admit I did struggle to wrap my head around all the tokens the first time I was going through the documentation. Nonetheless, all that is left for the extraction part is to call the activities endpoint and store the data into a `pandas` dataframe. Note that using this endpoint might be sufficient for you if you are not interested in the splits data, which is basically 1km aggregations containing distance, average speed, elevation difference, moving and elapsed time. This is how you would get it.

```python
def get_activities(access_token: str, per_page: int = 200, max_pages: int = 1) -> pd.DataFrame:
    data = []
    for page in range(1, max_pages + 1):
        page_data = get_activities_page(access_token, per_page, page)
        if len(page_data) == 0:
            break
        data.extend(page_data)
    return pd.json_normalize(data)

def get_activities_page(access_token, per_page: int = 200, page: int = 1) -> list:
    activites_url = "https://www.strava.com/api/v3/athlete/activities"
    headers = {'Authorization': 'Bearer ' + access_token}
    params = {'per_page': per_page, 'page': page}
    data = requests.get(activites_url, headers=headers, params=params).json()
    return data
```

Since I want all the data I can get, I would also call the other endpoint. For that we need to extract the activities IDs from the `get_activities` dataframe and then loop over it. This is done in the `get_activities_full` function.

```python
def get_activities_full(access_token: str, ids: list[str]) -> pd.DataFrame:
    result = []
    for id in ids:
        activity = get_activity(access_token, id)
        result.append(activity)
    return pd.DataFrame(result)

def get_activity(access_token: str, id: str) -> dict:
    activites_url = f"https://www.strava.com/api/v3/activities/{id}"
    headers = {'Authorization': 'Bearer ' + access_token}
    data = requests.get(activites_url, headers=headers).json()
    return data
```

As for the manipulation part, things start to get a little bit more complicated. Sure parsing dates is easy and so is converting velocity units, however parsing the map coordinates requires a little bit of extra domain knowledge. Let's take a quick look at the code below first.

```python
def format_activities_full(df: pd.DataFrame) -> pd.DataFrame:  
    df = df.copy(deep=True)  
    df.set_index('id', inplace = True)
    df['datetime'] = pd.to_datetime(df['start_date_local'])
    df['distance'] =  df['distance'] / 1000
    df['pace'] = (df['moving_time'] / 60) / df['distance']
    df['average_speed'] = df['average_speed'] * 3600 / 1000        
    df['coordinates'] = df['map.summary_polyline'].apply(polyline.decode)
    df['coordinates'] = df['coordinates'].apply(format_coordinates)
    df['coordinates'] = df['coordinates'].apply(get_elevation)
    df = df[['name', 'datetime', 'total_elevation_gain', 'distance', 'average_speed', 'pace', 'moving_time', 'elapsed_time', 'coordinates']]    
    return df

def format_coordinates(coordinates: list[tuple]) -> list[dict]:
    return [{'latitude': c[0], 'longitude': c[1] } for c in coordinates]  

def get_elevation(coordinates: list[dict]) -> list[dict]:
    url = 'https://api.open-elevation.com/api/v1/lookup'
    data = {'locations': coordinates}
    response = requests.post(url, json=data).json()
    return response['results']
```
Strava encodes the map coordinates with Google's [polyline algorithm](https://developers.google.com/maps/documentation/utilities/polylinealgorithm), luckily for us there is a handy python library called `polyline` made to handle just that. One other thing to notice is that we are further changing the format of the polyline from a list of tuples to list of dictionaries. That is because this format is required by [open elevation API](https://api.open-elevation.com/), unsurprisingly to get the elevation for given coordinates. This information will be useful later when we are visualizing the elevation profile of the track. 

Putting it all together, this is how the code above should be ordered. The function below also includes some extra time and distance filtering in order to prevent unnecessary api calls and keep only the relevant activities in the dataframe.

```python
def extract_full(client_id, client_secret, refresh_token):
    access_token = get_access_token(client_id, client_secret, refresh_token)
    activities = get_activities(access_token)
    activities['datetime'] = pd.to_datetime(activities['start_date_local'])
    italy = activities[(activities["datetime"] > "2022-06-16") & (activities["datetime"] < "2022-06-25")]
    italy = italy.query('distance > 5000')
    activities_full = get_activities_full(access_token, italy['id'])
    return format_activities_full(activities_full)
```

Now all that remains is to run the function, for example in python interactive mode. My file is named `strava.py`, therefore I would run `py -i strava.py`. Finally, we export the dataframe to a JSON file with `df.to_json('hikes_final.json', orient='records')` so that we can then easily fetch the file with javascript.

Alright that's it for the track data. In the next article I will take a look at all of the magic that can be done with geographic information systems (GIS). In other words, we will be modelling the basic building blocks for terrain generation in javacript. See you then!

