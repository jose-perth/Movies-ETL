# Movies-ETL

## Overview of the Analysis

This project is the eighth weekly challenge of the Data Science Bootcamp. It allows us to put into practice and showcare the skills learned in Module 8 of the bootcamp.

### Purpose

In this project we sequentially refactor the code created through Module 8 into a more complete one-click process to perform the ETL process and add the final data into a PostgreSQL database.

## Results

Input data files used:

- [wikipedia-movies.json](./Resources/wikipedia-movies.json)
The following files are too large to be uploaded. However, they can be obtained from obtained from [Kaggle](https://www.kaggle.com/rounakbanik/the-movies-dataset/download)
- movies_metadata.csv
- ratings.csv 

The final PostgreSQL database is located in a local machine.

### Deliverable 1: Write an ETL Function to Read Three Data Files 

The Jupyter Notebook created for this deliverable is:

[ETL_function_test.ipynb](./ETL_function_test.ipynb)

The aim of the deliverable is to create a function that would load the 3 files passed to it and return 3 DataFrames.

```python
# 1. Create a function that takes in three arguments;
# Wikipedia data, Kaggle metadata, and MovieLens rating data (from Kaggle)
def extract_transform_load(wiki_file, kaggle_file, ratings_file):
    # 2. Read in the kaggle metadata and MovieLens ratings CSV files as Pandas DataFrames.
    kaggle_metadata = pd.read_csv(kaggle_file, low_memory = False)
    ratings = pd.read_csv(ratings_file, low_memory = False)
    
    # 3. Open and read the Wikipedia data JSON file.
    with open(wiki_file, mode = 'r') as file:
        wiki_movies_raw = json.load(file)
    
    # 4. Read in the raw wiki movie data as a Pandas DataFrame.
    wiki_movies_df = pd.DataFrame(wiki_movies_raw)
    
    # 5. Return the three DataFrames
    return wiki_movies_df, kaggle_metadata, ratings
```

### Deliverable 2: Extract and Transform the Wikipedia Data

The Jupyter Notebook created for this deliverable is:

[ETL_clean_wiki_movies.ipynb](./ETL_clean_wiki_movies.ipynb)

This deliverable takes deliverable 1 one extra step on the ETL process and includes the extraction, and transformation to the Wikipedia data file.

Some items to point out from the code are:

- A ```try-except``` block is used to catch errors while extracting the IMDd IDs.

```python
# 6. Write a try-except block to catch errors while extracting the IMDb ID using a regular expression string and
    #  dropping any imdb_id duplicates. If there is an error, capture and print the exception.
    try:
        wiki_movies_df['imdb_id'] = wiki_movies_df['imdb_link'].str.extract(r'(tt\d{7})')
        wiki_movies_df.drop_duplicates(subset='imdb_id', inplace=True)
    except Exception as e: 
        print(e)
```
- List comprehension is used to keep columns with a high number of non-null values (>90%)

```python
    #  7. Write a list comprehension to keep the columns that don't have null values from the wiki_movies_df DataFrame.
    PERCENT_NULL = 0.9      # 0..1 percent of null values in columns to remove it
    wiki_columns_to_keep = [column for column in wiki_movies_df.columns if wiki_movies_df[column].isnull().sum() < len(wiki_movies_df)*PERCENT_NULL]
    wiki_movies_df = wiki_movies_df[wiki_columns_to_keep]
```

- Regular expressions are used to match elements for certain columns.

```python
    # 10. Write a regular expression to match the six elements of "form_one" of the box office data.
    form_one =  r'\$\s*\d+\.?\d*\s*[mb]illi?on'

    # 11. Write a regular expression to match the three elements of "form_two" of the box office data.
    form_two = r'\$\s*\d{1,3}(?:[,\.]\d{3})+(?!\s[mb]illion)'
```

### Deliverable 3: Extract and Transform the Kaggle Data

The Jupyter Notebook created for this deliverable is:

[ETL_clean_kaggle_data.ipynb](./ETL_clean_kaggle_data.ipynb)

This deliverable built on top of deliverable 2. We add the extraction and transformation of the Kaggle metadata and merged it to the Wikipedia metadata.

The data was merged using the merge function:

```python
    # 3. Merged the two DataFrames into the movies DataFrame.
    movies_df = pd.merge(wiki_movies_df, kaggle_metadata, on='imdb_id', suffixes=['_wiki','_kaggle'])
```

The ratings data was also processed and added to the movies_df dataframe to create a movies_with_ratings_df dataframe.

```python
    # 9. Transform and merge the ratings DataFrame.
    #simplify ratings df
    rating_counts = ratings.groupby(['movieId','rating'], as_index=False).count()\
                .rename({'userId':'count'},axis=1)\
                .pivot(index='movieId', columns='rating', values='count')
    #rename columns
    rating_counts.columns = ['rating_'+str(col) for col in rating_counts.columns]
    # merge ratings to movies_df
    movies_with_ratings_df = pd.merge(movies_df, rating_counts, left_on='kaggle_id', right_index=True, how='left')
    movies_with_ratings_df[rating_counts.columns]=movies_with_ratings_df[rating_counts.columns].fillna(0)
```

### Deliverable 4: Create the Movie Database

The Jupyter Notebook created for this deliverable is:

[ETL_create_database.ipynb](./ETL_create_database.ipynb)

This deliverable adds to the code in deliverable 3 by loading the movies_df and ratings data into 2 separate tables in the PostgreSQL database. The Ratings table needs to be dropped from within pgAdmin before running the code.  This could be changed in future modifications.

Part of the code to load the data follows.  Loading to the *rating* table is done in chunks and elapsed time is tracked to ensure the SQL server is responding appropriately:

```python
 db_string = f'postgres://postgres:{db_password}@127.0.0.1:5432/movie_data'
    engine=create_engine(db_string)
    # load movies_DF
    movies_df.to_sql(name='movies', con=engine, if_exists='replace')
# import ratings CSV slowly and load to SQL db
    rows_imported = 0
    start_time = time.time()
    for data in pd.read_csv(ratings_file, chunksize=1000000):
        # print range of rows being imported
        print(f'importing rows {rows_imported} to {rows_imported + len(data)}...', end='')  # do no end in new line
        data.to_sql(name='ratings', con=engine, if_exists='append')
        rows_imported += len(data)

        print(f'Done. {time.time() - start_time} total seconds elapsed.')
```

We can confirm that the data was loaded into the PostgreSQL database by running queries on it.

![Number of movies](./Resources/movies_query.png)

![Ratings count](./Resources/rating_query.png)

**Note:**  the numer of total movies on the table (6,075) is higher than number expected on the challenge (6,052). It's a difference of 0.23% due to the criteria used to remove TV shows.

## Summary

This challenge walked us through the development of an ETL pipeline. By adding complexity and steps to each subsequent iteration, a final one-click function was developed to convert the data files into an useful SQL database.