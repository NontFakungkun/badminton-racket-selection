> I personally struggle to find a suitable badminton racket and have to go through a rigorous amount of research to reach a conclusion. I wish there was a one-stop resource that could help me decide which racket is the perfect match based on my preferences. The resource should be comprehensive (include most of the popular rackets) and user-friendly.
>
> Also as a bonus, I would like to find what factors make rackets suitable for each skill level and play style, what affects racket price, and how pro athletes pick their racket.


# Objectives of this project

- [x] Establish a badminton racket selection-aiding resource by clustering rackets into 4 quadrants controlled by filters to help users find their ideal rackets.

- [x] Aggregate values group by skill_level, and playstyle to find pattern of rackets suitable for different player types.

- [x] Find price correlation to see what is associated with price e.g. Material bin and regression

- [x] Pro athlete racket choices.

 

# Data Acquisition 

We combine Yonex badminton racket data scraped from 2 different websites which have datasets that together turn into a comprehensive dataset for the topic.

https://badmintongeardatabase.siterubix.com/badminton-rackets-list/

https://www.badmintonmag.com/yonex-badminton-rackets-list/

Due to the data in these sources are in different format, the scraping method needs to be bit different.

The data in the first source is in table format.
```
url= 'https://badmintongeardatabase.siterubix.com/badminton-rackets-list/'
url2 = 'https://www.badmintonmag.com/yonex-badminton-rackets-list/'

page = requests.get(url)
soup = BeautifulSoup(page.text,'html')

table = soup.find_all('table')[1]
headings = table.find_all('th')[:-1]
heading_table = [ hd.text.strip() for hd in headings ]
df1 = pd.DataFrame(columns = heading_table)

data_details = table.find_all('tr')[1:]
for row in data_details:
    row_data = row.find_all('td')[:-1]
    data_table = []
    for data in row_data:
        data_chunk = data.text.strip()
        if '\n' in data_chunk:
            data_table.append(', '.join(data_chunk.split('\n')))
        else:
            data_table.append(data_chunk)
    df1.loc[len(df1)] = data_table
```

The second source is a bit more raw, the data for each racket is in linear form so we need to acquire all racket name first before combining with their details.
```
page = requests.get(url2)
soup = BeautifulSoup(page.text,'html')

table = soup.find_all('div', class_ = "entry-content entry clearfix")[0]

heading = table.find_all('ul')[0]
heading_list = [ hd.text.strip().split(':')[0] for hd in heading ]
heading_list.insert(0, 'Full Name')
heading_list[-2] = ' '.join(heading_list[-2].split(' ')[:2])
df2 = pd.DataFrame(columns = heading_list)

names = table.find_all('h3', class_ = 'wp-block-heading')
names = [ name.text.strip() for name in names ]

data_details = table.find_all('ul')
for index, row in enumerate(data_details):
    row_data = row.find_all('li')
    data_table = []
    for data in row_data:
        data = data.text.strip().split(':')[-1].strip()
        if '/' in data:
            data_table.append(', '.join([ item.strip() for item in data.split('/') ]))
        else:
            data_table.append(data)
    data_table_combine = [names[index]] + data_table
    df2.loc[len(df2)] = data_table_combine
```

In order to merge the 2 datasets, we need to 

* Standardise the racket name removing brand name
* Remove the overlapping field ‘Balance’, I decided to delete the one with missing values.

```
df1["Full Name"] = df1["Full Name"].apply(lambda x: ' '.join(x.split(' ')[1:]))
df1 = df1.drop(columns='Balance')
df_merged = df2.merge(df1, how='left', on='Full Name')
```


Here is the merged table preview



# Data Preparation
## Data Cleaning
We cleaned and manipulated the newly merged dataset in Python Pandas

``` 
# Remove columns with missing values
# Remove more overlapping columns 
# remove columns irrelevant to our objectives
df_merged = df_merged.drop(columns='Grip size')
df_merged = df_merged.drop(columns='Frame Shape')
df_merged = df_merged.drop(columns='Special Features')
df_merged = df_merged.drop(columns='Frame Material')
df_merged = df_merged.drop(columns='Shaft Material')
df_merged = df_merged.drop(columns='Best Strings')
df_merged = df_merged.drop(columns='Length (mm)')

# Remove rackets with missing spec and racket for kids
df_merged['Balance'].replace('', np.nan, inplace=True)
df_merged.dropna(subset='Balance', inplace=True)
df_merged = df_merged[:-1]

# Cleaning null values
df_merged = df_merged.fillna('')
df_merged['Players using'] = df_merged['Players using'].str.replace('–','')

# Rephrase values in ‘stiffness’ field
df_merged['Stiffness'] = df_merged['Stiffness'].str.replace(' shaft', '')

# Create ‘series’ field by extracting full names
df_merged['Series'] = df_merged['Full Name'].apply(lambda x: x.split(' ')[0])

df_merged.to_csv(r'C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_racket_dataset.csv', index = True)
```

Due to the dataset being small, I decided to use excel to further clean and manipulate the data as I can edit them in more details

* Setting id
* Format weird character for readability
* Use filter to identify error, and abnormality in data and address them
* Replace price field from AUD to Baht by calculating price from AUD (using rate AUD = 24.44 Baht referenced from Google on 24th June)
* Decrease variability of stiffness so the data is more consistent and reliable
* There are ‘size’ and ‘weight’ fields from two datasets, all of them are incomplete, so I will combine them in the same format to make it as complete as possible


Our dataset is now clean and standardised, the next step will be categorising rackets for each skill level and play style by using text detection to extract information from ‘suitable_for’ field.
```
df_cleaned = pd.read_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_racket_dataset_cleaned.csv")

df_cleaned['skill_level'] = df_cleaned['suitable_for'].apply(
    lambda x: 'Beginner' if 'beginner' in x.lower() or 'beginners' in x.lower() or 'recreational' in x.lower() else (
        'Intermediate' if 'intermediate' in x.lower() else (
            'Advanced' if 'advanced' in x.lower() else 'null'
        )
    )
)

df_cleaned['play_style'] = df_cleaned['suitable_for'].apply(
    lambda x: 'All-round' if 'allround' in x.lower() else (
        'Offense' if 'attacking ' in x.lower() or 'offense' in x.lower() or 'power' in x.lower() else (
            'Defense' if 'defense' in x.lower() or 'control' in x.lower() else (
                'Technique' if 'technique' in x.lower() else 'All-round'
            )
        )
    )
)

df_cleaned.to_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_racket_dataset_cleaned2.csv", index=False)
```
However, there are some fields that my code can’t detect so I put the rest in manually in excel based on my comprehension.
 

## Data Normalisation

Currently, ‘size’, ‘weight’, and ‘players_using’ fields, to make them usable, I will create new tables and link them to the main table with racket_id.

The new tables are:
* badminton size data with two primary key
* Badminton Player data table id, name, gender, play_type, racket_used

```
df_cleaned = df_cleaned.fillna('')
df_size = pd.DataFrame(columns = ['racket_id', 'racket_name', 'size', 'weight'])
df_player = pd.DataFrame(columns = ['player_id', 'player_name', 'gender', 'play_type', 'racket_used'])

for i in range(len(df_cleaned)):
    size_list = df_cleaned['size'][i].split(', ')
    for index, size in enumerate(size_list):
        weight = df_cleaned['weight'][i].split(', ')[index].split(' ')[0].strip()
        df_size.loc[len(df_size)] = [df_cleaned['racket_id'][i], df_cleaned['full_name'][i], size, weight]

for i in range(len(df_cleaned)):
    players_list = df_cleaned['players_using'][i].split(', ')
    for player in players_list:
        if player:
            df_player.loc[len(df_player)] = [len(df_player)+1, player, '', '', df_cleaned['racket_id'][i]]

df_size.to_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_racket_size_dataset.csv", index=False)
df_player.to_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_player_racket_dataset.csv" index=False)
```
The player table still needs to be finalised as gender and play type is not given but I can put them in manually since I’m a badminton fan.


As result, these are the normalised and cleaned tables ready for further analysis

### Main Racket Table
pic

### Racket Size Table
pic

### Player Table
pic

# Entity Relationship Diagram

```
Table size {
  racket_id integer [primary key]
  racket_name varchar
  size varchar
  weight_g int
}

Table racket {
  racket_id integer [primary key]
  full_name varchar
  series varchar
  balance varchar
  stiffness varchar
  balance_rating integer
  stiffness_rating integer
  material varchar
  suitable_for text
  skill_level varchar
  play_style varchar
  price_baht float

}

Table athlete {
  player_id integer [primary key]
  player_name varchar
  gender varchar
  play_type integer
  racket_used int
}

ref: size.racket_id > racket.racket_id
ref: racket.racket_id < athlete.racket_used
```

## Data Dictionary

### Main Racket Table
* racket_id: ID of badminton racket
* full_name: Badminton racket full name
* series: Badminton racket series
* balance: The balance of badminton racket between head and shaft
* stiffness: Badminton racket shaft stiffness
* material: Material used in making of badminton racket
* suitable_for: Whom the badminton racket is suitable for
* skill_level: Recommended skill level to play with this badminton racket
* play_style: Badminton racket play style suitability
* price_baht: Badminton racket price in Baht

### Racket Size Table
* racket_id: ID of badminton racket (Linked to Main Racket Table)
* racket_name: Badminton racket full name (Linked to Main Racket Table)
* size: Badminton racket size with unit U (light 5U - U heavy)
* weight_g: Badminton racket weight

### Player Table
* player_id: ID of badminton player
* player_name: Badminton player full name
* gender: Badminton player gender
* play_type: Badminton player play type (singles or doubles)
* racket_used: Racket that the player used (Linked to Main Racket Table)



# Exploratory Data Analysis

Examine mean values of badminton racket spec and price based on skill level, and play style to find pattern of rackets suitable for different player types.

To make balance and stiffness of the racket calculable, I decided to rate them from 1-5


	

1

	

2

	

3

	

4

	

5




Balance

	

Head-light

	

Slightly Headlight

	

Medium

	

Slightly head-heavy

	

Head-heavy 




Stiffness

	

Very flexible

	

flexible

	

Even balanced

	

Stiff 

	

Very stiff
```
df_racket = pd.read_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_racket_dataset_cleaned2.csv")
df_size = pd.read_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_racket_size_dataset_cleaned.csv")
df_player = pd.read_csv(r"C:\Users\Nont\Desktop\Data analytics\Projects\Badminton Analysis\badminton_player_racket_dataset_cleaned.csv")

df_racket.groupby('skill_level').agg(
    {'balance_rating': ['count', 'mean', 'min', 'max', 'median'],
    'stiffness_rating': ['count', 'mean', 'min', 'max', 'median'],
    'price_baht': ['count', 'mean', 'min', 'max', 'median']
    })

df_racket.groupby('play_style').agg(
    {'balance_rating': ['count', 'mean', 'min', 'max', 'median'],
    'stiffness_rating': ['count', 'mean', 'min', 'max', 'median'],
    'price_baht': ['count', 'mean', 'min', 'max', 'median']
    })
```


For spec analysis, I decided to only examine the mean from all aggregated values as other values don't provide any useful insight. 
```
df_skill_spec = df_racket.groupby('skill_level')[['balance_rating', 'stiffness_rating']].mean(numeric_only=True).sort_values("balance_rating",ascending=True)

df_skill_spec_plot = df_skill_spec.T.plot(kind='bar', figsize=(12, 8))

# Title and labels
df_skill_spec_plot.set_title('Badminton Racket Spec Plot Grouped by Skill Level')
df_skill_spec_plot.set_xlabel('Rating Type')
df_skill_spec_plot.set_ylabel('Rating')

# Rotating labels on x-axis 
df_skill_spec_plot.set_xticklabels(['Balance Rating','Stiffness Rating'], rotation=0)

# legend
df_skill_spec_plot.legend(title='Skill Level', loc='upper left')

df_play_spec = df_racket.groupby('play_style')[['balance_rating', 'stiffness_rating']].mean(numeric_only=True).sort_values("balance_rating",ascending=True)

df_play_spec_plot = df_play_spec.T.plot(kind='bar', figsize=(12, 8))

# Title and labels
df_play_spec_plot.set_title('Badminton racket spec plot grouped by play style')
df_play_spec_plot.set_xlabel('Play Style')
df_play_spec_plot.set_ylabel('Rating')

# Rotating labels on x-axis
df_play_spec_plot.set_xticklabels(['Balance Rating','Stiffness Rating'], rotation=0)

# legend
df_play_spec_plot.legend(title='Play Style', loc='upper left')
```
              

The balance and stiffness of the racket increase proportionally to skill level. Meanwhile, the attacking racket is head-heavy, the all-round racket are balanced, and the defensive racket is head-light. The racket for training has understandably head-heaviest balance to help players build strength and improve their technique. There is not much difference in term of stiffness except that the offensive racket has slightly higher stiffness than others.




For the price analysis, I used box plot to display a comprehensive ranges of price for different groups of racket.
```
desired_order = ['Beginner', 'Intermediate', 'Advanced']
df_racket['skill_level'] = pd.Categorical(df_racket['skill_level'], categories=desired_order, ordered=True)
df_racket = df_racket.sort_values('skill_level')

df_racket.boxplot(column='price_baht', by='skill_level')
plt.title('Badminton Racket Price (Baht) Boxplot Grouped by Skill Level')
plt.suptitle('')
plt.xlabel('Skill Level')
plt.ylabel('Price (Baht)')

plt.show()


desired_order = ['Beginner', 'Intermediate', 'Advanced']
df_racket['skill_level'] = pd.Categorical(df_racket['skill_level'], categories=desired_order, ordered=True)
df_racket = df_racket.sort_values('skill_level')

# Drop Training group as there's no price provided hence no box
filtered_df = df_racket[df_racket['play_style'] != 'Training']

filtered_df.boxplot(column='price_baht', by='play_style')
plt.title('Badminton Racket Price (Baht) Boxplot Grouped by Play style')
plt.suptitle('')
plt.xlabel('Play style')
plt.ylabel('Price (Baht)')

plt.show()
```
            

The price for beginner rackets are relatively low while the intermediate and advanced rackets have similar price with intermediate rackets having larger price range.

For play style, the result is getting interesting, all round racket price range is huge covering both cheap and premium racket, while the defense racket is slightly higher. However, the offensive racket has the highest price with lowest price range despite having multiple entries in the data. This means offensive rackets are priced similarly high for some reason.



Examine price correlation to see what is associated with price. 

Regression: We found a small positive relationship in the linear regression analysis for both the Balance-Price and Stiffness-Price relationships. However, the regression results may not be reliable due to the limited number of scatter points from the small data sample.

> [!NOTE]
> to make regression doable, I temporarily removes row with null price, so they will only contain 34 from 53 rackets

```
df_racket_temp = df_racket.dropna(subset="price_baht")

X_train, X_test, y_train, y_test = train_test_split(df_racket_temp[['balance_rating']], df_racket_temp['price_baht'], test_size=0.2, random_state=42)

model = LinearRegression()
model.fit(X_train, y_train)


y_pred = model.predict(X_test)


plt.scatter(X_test, y_test, color='blue', label='Actual')
plt.plot(X_test, y_pred, color='red', linewidth=2, label='Predicted')
plt.xlabel('Racket Balance')
plt.ylabel('Price (Baht)')
plt.legend()
plt.title('Linear Regression Analysis of Badminton Racket Balance and Price')
plt.show()


X_train, X_test, y_train, y_test = train_test_split(df_racket_temp[['stiffness_rating']], df_racket_temp['price_baht'], test_size=0.2, random_state=42)

model = LinearRegression()
model.fit(X_train, y_train)

y_pred = model.predict(X_test)

plt.scatter(X_test, y_test, color='blue', label='Actual')
plt.plot(X_test, y_pred, color='red', linewidth=2, label='Predicted')
plt.xlabel('Racket Stiffness')
plt.ylabel('Price (Baht)')
plt.legend()
plt.title('Linear Regression Analysis of Badminton Racket Stiffness and Price')
plt.show()
```
             




Material bin: Horizontal bar plot is used to show which materials are being used in badminton at different prices. The logic is that I find the average price of badminton rackets containing those materials and use that as plot value. This is because one racket is made of multiple materials so it will be difficult to compare material by racket.
```
df_material_price = df_racket.groupby('material')[['price_baht']].mean(numeric_only=True).sort_values("price_baht",ascending=True)

# Drop rackets with no defined price
df_material_price = df_material_price.dropna()

# Logic for finding average racket prices containing each materials
material_sum_price = dict()
for materials, row_data in df_material_price.iterrows():    
    price = row_data['price_baht']
    for mat in materials.split(', '):
        if mat not in material_sum_price:
            material_sum_price[mat] = [price, 1]
        else:
            material_sum_price[mat][0] += price
            material_sum_price[mat][1] += 1
material_price_df = pd.DataFrame(columns = ['Material', 'avg_price_contained'])
for key, value in material_sum_price.items():
    material_price_df.loc[len(material_price_df)] = [key, round(value[0]/value[1],2)]
material_price_df = material_price_df.sort_values('avg_price_contained', ascending=False)

# Plot
material_price_df.plot(kind='barh', title='Avg price of badminton rackets containing the material', 
                       x='Material', ylabel='Materials', xlabel='Price (Baht)', legend=False)

As a result, steel and aluminium have the lowest average racket prices implying the metals used in cheap rackets while more fancy names lke Neo CS Carbon Nanotube, Rexil Fiber and Namd have the highest average racket prices implying they are used in premium rackets.

This chart can be useful when examining the materials of the rackets you're interested in. If a racket contains a particular material and is priced lower than the average racket with that material, it could mean a good deal for you.
```



Pro athlete choices 
```
# Combine racket and player table on racket ID
df_player_racket = df_player.merge(df_racket, left_on='racket_used', right_on='racket_id')

# Get mean values of badminton racket balance and stiffness based on athlete gender and play type
df_mean_ratings = df_player_racket.groupby(['play_type', 'gender'])[['balance_rating', 'stiffness_rating']].mean(numeric_only=True).reset_index()

# Heatmap Plot
rating_pivot = df_mean_ratings.pivot(index='play_type', columns='gender', values=['balance_rating', 'stiffness_rating'])
rating_pivot.columns = rating_pivot.columns.swaplevel(0, 1)
rating_pivot = rating_pivot.sort_index(axis=1)
new_labels = [col[0] + ' ' + ' '.join(col[1].split('_')).title() for col in rating_pivot.columns.values]

plt.figure(figsize=(10, 6))

sns.heatmap(rating_pivot, annot=True, cmap='coolwarm')
plt.xticks(ticks=range(len(new_labels)), labels=new_labels, rotation=30)
plt.xlabel(None)
plt.ylabel(None)
plt.title('Heatmap of Average Racket Balance and Stiffness Ratings by Play Type and Gender', pad=20)
plt.show()
```

I found that the table display already conveys the findings quite well, so I enhanced it with a heatmap using cool and warm colors to further emphasize the differences in racket specifications for each category.

Averagely, female athlete use racket with balance around 2.14 (slightly head-light) with stiffness of 3.64 (between medium to stiff shaft), while male athlete use racket with balance around 3.71 (almost slightly head-heavy) with stiffness of 4.35 (between stiff to very stiff shaft).

Looking into the play types, Men doubles players prefer a much head-heavier (4.4) and stiffer (4.6) racket, while Men singles players use rackets with spec similar to average male athletes. (3.63 for balance rating and 4.38 for stiffness). This is probably due to the fact that Men's doubles only need to cover around half of the court and require less energy in a game. Therefore, the athlete decided to use a head-heavy and stiff racket for more powerful shots. However, that kind of play style will exhaust singles players before their games end. However, it seems the pattern seems to be different for female athletes. It turned out that female singles players use rackets with heavier heads than female doubles. Again, I presumed this is affected by the play style. Offensive play style is not so favourable in women’s doubles so they might prefer rackets with lighter head.

Another interesting result is the racket choice of mixed double players. Male mixed doubles players use the balanced racket rather than head-heavy like other male play types. Their rackets also have a stiff rating of 4 which is less than the male average. For female mixed doubles players, their racket balance rating is 1.8 (slightly head-light) and stiffness rating of 3.4 which is both lower than female singles racket. The reason is the faster tempo in mixed doubles games. The athlete needs to use head-lighter rackets which have more racket speed to catch up with the pace.

As an insight, while the racket balance varies for different gender and play type, the stiffness of their rackets is no less than 3. Even the minimum mean for stiffness is 3.4. This means the athletes prefer stiff rackets over flexible ones. They do not require the help of a flexible racket to push the shuttle across.

In conclusion, not all athletes opted for power, they might prefer speed and control based on the nature of the play types. The head-heavy or stiffer racket is not better than the head-light or flexible racket and vice versa. So, your racket choice should depend on your play style and support you need from the racket.

 
> [!NOTE]
> there’s only 1 entry of female doubles players so the result might not be fully representative.
 

# Interactive Racket Selection Tool

Now come the highlight. I intended to have a scatter plot reflecting two main spec of the racket along with having multiple filters to help users select their suitable badminton racket. Each racket is coloured by series and shaped by skill level for visibility.

The link to interactive chart in Tableau Public: https://public.tableau.com/views/DiscoverYourRacketYonexBadmintonRacketSelectionTool/Dashboard1?:language=en-US&:sid=&:display_count=n&:origin=viz_share_link


