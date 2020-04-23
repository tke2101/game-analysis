

```python
from IPython.display import HTML
import pandas as pd
from datetime import datetime

%matplotlib inline
from matplotlib import pyplot as plt
```


```python
HTML('''<script>
code_show=true; 
function code_toggle() {
 if (code_show){
 $('div.input').hide();
 } else {
 $('div.input').show();
 }
 code_show = !code_show
} 
$( document ).ready(code_toggle);
</script>
The raw code for this IPython notebook is by default hidden for easier reading.
To toggle on/off the raw code, click <a href="javascript:code_toggle()">here</a>.''')
```




<script>
code_show=true; 
function code_toggle() {
 if (code_show){
 $('div.input').hide();
 } else {
 $('div.input').show();
 }
 code_show = !code_show
} 
$( document ).ready(code_toggle);
</script>
The raw code for this IPython notebook is by default hidden for easier reading.
To toggle on/off the raw code, click <a href="javascript:code_toggle()">here</a>.




```python
df = pd.read_csv('data/telemetry.csv')
df['timestamp'] = pd.to_datetime(df['timestamp'])

version_A = df[df.game_version=='A']
version_B = df[df.game_version=='B']

total_users_A = len(pd.unique(version_A['user_id']))
total_users_B = len(pd.unique(version_B['user_id']))

level1_date_version_A = df[((df.event_name=='LevelStart')&((df.current_level==1)&(df.game_version=='A')))][['user_id', 'session_id','timestamp']]
level1_date_version_B = df[((df.event_name=='LevelStart')&((df.current_level==1)&(df.game_version=='B')))][['user_id', 'session_id','timestamp']]


level_starts_version_A = version_A[version_A.event_name=='LevelStart']
level_starts_version_B = version_B[version_B.event_name=='LevelStart']

level1_date_version_A['first_play_date'] = level1_date_version_A['timestamp'].apply(lambda x: datetime.date(x))
level1_date_version_B['first_play_date'] = level1_date_version_B['timestamp'].apply(lambda x: datetime.date(x))

level1_date_version_A = level1_date_version_A.groupby('user_id').min().reset_index()
level1_date_version_B = level1_date_version_B.groupby('user_id').min().reset_index()

level1_date_version_A.drop(['timestamp', 'session_id'], axis=1, inplace=True)
level1_date_version_B.drop(['timestamp', 'session_id'], axis=1, inplace=True)

retention_version_A = level_starts_version_A.merge(level1_date_version_A, on='user_id', how='inner')
retention_version_B = level_starts_version_B.merge(level1_date_version_B, on='user_id', how='inner')

retention_version_A['day_number'] = (retention_version_A['timestamp'].apply(lambda x: datetime.date(x)) - retention_version_A['first_play_date']).dt.days
retention_version_B['day_number'] = (retention_version_B['timestamp'].apply(lambda x: datetime.date(x)) - retention_version_B['first_play_date']).dt.days

#take a look at users who would have had a full 7 days only to unbias
#retention_version_A = retention_version_A[retention_version_A.first_play_date<=datetime.strptime('2019-10-28', '%Y-%m-%d').date()]
#retention_version_B = retention_version_B[retention_version_B.first_play_date<=datetime.strptime('2019-11-08', '%Y-%m-%d').date()]

day_retention_A = retention_version_A.groupby('day_number')['user_id'].nunique().reset_index(name='retention')
day_retention_A['retention'] = day_retention_A['retention']/total_users_A

day_retention_B = retention_version_B.groupby('day_number')['user_id'].nunique().reset_index(name='retention')
day_retention_B['retention'] = day_retention_B['retention']/total_users_B

```

# High level summary

Similar numbers of users installed either version A or version B of the game (198 for A, 188 for B). Overall, version B shows higher long term retention than A (up to 7 days). Engagement (sessions per user and level plays per user) was also higher. 

Monetization was lower for version B, but the time period users played for was also much shorter, so we can't make a like-for-like comparison. 

#### Recommendation: Chose version B.

### KPIs


#### Daily Active Users


```python
retention_version_A['dt'] = retention_version_A['timestamp'].apply(lambda x: datetime.date(x))
retention_version_B['dt'] = retention_version_B['timestamp'].apply(lambda x: datetime.date(x))


dau_A = retention_version_A.groupby('dt').user_id.nunique().reset_index(name='num_users')
dau_B = retention_version_B.groupby('dt').user_id.nunique().reset_index(name='num_users')

plt.plot(dau_A['dt'], dau_A['num_users'], label='version A')
plt.plot(dau_B['dt'], dau_B['num_users'], label='version B')
plt.legend()
plt.xlabel('Date')
plt.ylabel('Number of users')
plt.title('DAU vs. date')
plt.xticks(rotation=45)
plt.show()
```


![png](output_6_0.png)


#### Retention

Day 1 retention rates are similar for the two versions, but then diverge over longer periods. From day 2 onward version B shows higher retention. 


```python
plt.plot(day_retention_A['day_number'][:8], day_retention_A['retention'][:8], label='version A')
plt.plot(day_retention_B['day_number'][:8], day_retention_B['retention'][:8], label='version B')
plt.legend()
plt.xlabel('Days after install')
plt.ylabel('Retention')
plt.title('Retention Rates by Day')
plt.show()
```


![png](output_8_0.png)


### Engagement

Sessions per day per user:


```python
sessions_per_day_A = retention_version_A.groupby('day_number')[['session_id', 'user_id']].nunique().reset_index()
sessions_per_day_B = retention_version_B.groupby('day_number')[['session_id', 'user_id']].nunique().reset_index()
sessions_per_day_A.columns=['day_number', 'sessions', 'users']
sessions_per_day_B.columns=['day_number', 'sessions', 'users']

sessions_per_day_A['sessions_per_user'] = sessions_per_day_A['sessions']/sessions_per_day_A['users']
sessions_per_day_B['sessions_per_user'] = sessions_per_day_B['sessions']/sessions_per_day_B['users']

plt.plot(sessions_per_day_A['day_number'][:8], sessions_per_day_A['sessions_per_user'][:8], label='version A')
plt.plot(sessions_per_day_B['day_number'][:8], sessions_per_day_B['sessions_per_user'][:8], label='version B')
plt.legend()
plt.xlabel('Days after install')
plt.ylabel('Sessions per User')
plt.title('Engagement by Days After Install')
plt.show()
```


![png](output_10_0.png)


### Funnel

Users in versions A and B dropped off at roughly similar rates by level, though version B users made it slightly farther. 


```python
level_starts_A = version_A[version_A.event_name=='LevelStart'].groupby('current_level')['user_id'].nunique().reset_index(name='num_users')
level_starts_B = version_B[version_B.event_name=='LevelStart'].groupby('current_level')['user_id'].nunique().reset_index(name='num_users')

plt.plot(level_starts_A['current_level'], level_starts_A['num_users'], label='version A')
plt.plot(level_starts_B['current_level'], level_starts_B['num_users'], label='version B')
plt.legend()
plt.xlabel('Level')
plt.ylabel('Number of users')
plt.title('Number of Users Reaching Each Level')
plt.show()


```


![png](output_13_0.png)


### Monetization


```python

```

#### Open Questions

What was the purpose of the intro, which only appeared for version A? 

What changes were made to the levels between versions? The pass rates look roughly similar, but I would like to know if something else changed that influenced retention. 



### Next steps

I would recommend following users in variant B for another 11 days so that the time window used to compare the two versions is equivalent. This will allow us to compare monetization rates over equivalent periods and see if there is a difference between the two versions. 

#### Telemetry improvements

We could use some improvements to our tracking to better understand how users play the game: 
    - Enhanced purchase events. We currently only know how much players spend on currency and at what level, not what this money is spent on. What types of boosters are they buying? From this we could better tell if we're offering the right selection of boosters and which ones are most useful for which level. 
    - An event telling us when users uninstall the game from their phone. This would give a clear indication they are unhappy with the game and have completely stopped playing. 
    - Logging the type of mobile device would be very useful. Now we don't know if there are any differences between ios and android, and between phone and tablet. 


```python

```
