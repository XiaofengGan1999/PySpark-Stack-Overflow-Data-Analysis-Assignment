# PySpark-Stack-Overflow-Data-Analysis-Assignment
PySpark Stack Overflow Data Analysis Assignment
This repository contains a complete solution for analyzing a subset of the Stack Overflow data dump using Apache Spark and PySpark. The primary objective of this assignment is to study the Programming Language Popularity Over Time. The analysis examines the relative popularity of the top 10 programming languages over the past five years, computing per-quarter post counts, percentage contributions, and quarter-over-quarter growth rates. It then visualizes these trends and summarizes the fastest growing and declining languages.

Table of Contents
Overview

Dataset Description

Project Setup

Data Acquisition

Code Overview

Extracting Language Tags

Filtering Data (Past Five Years)

Identifying Top 10 Languages

Creating Time-Series Data

Calculating Growth Rate

Visualization and Summary

How to Run

Dependencies

Notes

License

Overview
This assignment demonstrates how to use PySpark to:

Read and process Parquet files containing posts, users, comments, and tags.

Extract programming language tags from post data.

Filter posts based on a specific time period.

Use DataFrame operations and UDFs to manipulate and analyze data.

Utilize window functions for time-series analysis (quarter-over-quarter growth).

Generate summary statistics and visualizations.

Dataset Description
The dataset is a resampled subset of the Stack Overflow data dump. It includes:

posts.parquet: Contains questions and answers. Key columns include Id, PostTypeId, CreationDate, Score, Body, Title, Tags, OwnerUserId, ParentId, ViewCount, CommentCount, and ClosedDate.

users.parquet: Contains user information.

comments.parquet: Contains comments on posts.

tags.parquet: Contains tag information.

For this assignment, we primarily use posts.parquet.

Project Setup
This project is designed to run in Google Colab with PySpark. The setup includes:

Initializing a SparkSession:
We create a SparkSession to run our analysis. For example:

python
Copy
Edit
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("StackOverflow Analysis") \
    .master("local[*]") \
    .getOrCreate()
Note: In Colab, you can remove the .master("local[*]") line if needed.

Setting Up the Environment for GCS (Optional):
If accessing data from Google Cloud Storage (GCS), download the GCS connector jar and configure the Spark session accordingly.

Data Acquisition
Download and extract the dataset directly within Colab using the following commands:

bash
Copy
Edit
!wget https://storage.googleapis.com/rotman-data/so-small.zip
!unzip so-small.zip -d /content/so-small
After extraction, verify the files:

bash
Copy
Edit
!ls /content/so-small
Ensure that files such as posts.parquet, users.parquet, comments.parquet, and tags.parquet are present.

Code Overview
Below is a breakdown of the key steps in the analysis:

Extracting Language Tags
Objective: Extract individual programming language tags from the Tags column in posts.parquet.

Method:
Define a UDF (User-Defined Function) to parse the tag string (formatted as <python><java>, etc.) and extract languages using regular expressions.

python
Copy
Edit
def extract_tags(tags):
    if tags:
        return re.findall(r'<([^>]+)>', tags)
    else:
        return []

extract_tags_udf = F.udf(extract_tags)
posts_with_tags = posts_df.withColumn("languages", extract_tags_udf(F.col("Tags")))
posts_exploded = posts_with_tags.withColumn("language", F.explode("languages"))
Filtering Data
Objective: Limit the analysis to posts from the past five years.

Method:
Convert CreationDate to a date and filter for posts from 2018 onward.

python
Copy
Edit
posts_exploded = posts_exploded.withColumn("CreationDate", F.to_date("CreationDate"))
posts_recent = posts_exploded.filter(F.col("CreationDate") >= F.lit("2018-01-01"))
Identifying Top 10 Languages
Objective: Find the top 10 programming languages by overall post count.

Method:
Group by language and count posts, then sort and limit to the top 10.

python
Copy
Edit
top_languages = posts_recent.groupBy("language") \
    .count() \
    .orderBy(F.desc("count")) \
    .limit(10)
top_languages_list = [row['language'] for row in top_languages.collect()]
posts_top = posts_recent.filter(F.col("language").isin(top_languages_list))
Creating Time-Series Data
Objective: Create a time-series analysis with quarterly granularity.

Method:
Create a new column quarter in the format YYYY-Q# by extracting year and quarter from CreationDate.

python
Copy
Edit
posts_top = posts_top.withColumn("year", F.year("CreationDate")) \
    .withColumn("q", F.quarter("CreationDate")) \
    .withColumn("quarter", F.concat_ws("-Q", F.col("year"), F.col("q")))
Aggregation:
Compute the post count per language per quarter and calculate the percentage of posts for each language relative to the total posts in that quarter.

python
Copy
Edit
agg_df = posts_top.groupBy("language", "quarter").agg(F.count("*").alias("post_count"))
total_by_quarter = agg_df.groupBy("quarter").agg(F.sum("post_count").alias("total_posts"))
agg_df = agg_df.join(total_by_quarter, on="quarter", how="left") \
    .withColumn("percentage_of_total", F.round(F.col("post_count")/F.col("total_posts")*100, 2))
Calculating Growth Rate
Objective: Determine the quarter-over-quarter growth rate for each language.

Method:
Use window functions to calculate the previous quarter’s post count, then compute the percentage change.

python
Copy
Edit
windowSpec = Window.partitionBy("language").orderBy("quarter")
agg_df = agg_df.withColumn("prev_post_count", F.lag("post_count").over(windowSpec))
agg_df = agg_df.withColumn("growth_rate", 
                           F.when(F.col("prev_post_count").isNotNull(),
                                  F.round((F.col("post_count") - F.col("prev_post_count"))/F.col("prev_post_count")*100, 2))
                            .otherwise(None)
                           )
result_df = agg_df.select("language", "quarter", "post_count", "percentage_of_total", "growth_rate") \
    .orderBy("language", "quarter")
Visualization and Summary
Objective: Visualize trends and summarize growth.

Method:
Convert the result DataFrame to a Pandas DataFrame for plotting with Matplotlib.

python
Copy
Edit
import matplotlib.pyplot as plt
import pandas as pd

pd_df = result_df.toPandas()
pivot_df = pd_df.pivot(index="quarter", columns="language", values="percentage_of_total")
pivot_df = pivot_df.sort_index()

plt.figure(figsize=(12, 6))
for lang in pivot_df.columns:
    plt.plot(pivot_df.index, pivot_df[lang], marker='o', label=lang)
plt.xlabel("Quarter")
plt.ylabel("Percentage of Total Posts (%)")
plt.title("Programming Language Popularity Over Time")
plt.xticks(rotation=45)
plt.legend(title="Language", bbox_to_anchor=(1.05, 1), loc='upper left')
plt.tight_layout()
plt.show()
Growth Statistics:
Calculate average quarterly growth rates to list the top 5 fastest growing and top 5 fastest declining languages.

python
Copy
Edit
growth_stats = result_df.groupBy("language").agg(F.avg("growth_rate").alias("avg_growth_rate"))
fastest_growing = growth_stats.orderBy(F.desc("avg_growth_rate")).limit(5)
fastest_declining = growth_stats.orderBy("avg_growth_rate").limit(5)

print("Fastest Growing Languages:")
fastest_growing.show(truncate=False)

print("Fastest Declining Languages:")
fastest_declining.show(truncate=False)
How to Run
Launch Google Colab: Open a new notebook in Google Colab.

Upload and Extract Data:
Run the following cells to download and extract the dataset:

python
Copy
Edit
!wget https://storage.googleapis.com/rotman-data/so-small.zip
!unzip so-small.zip -d /content/so-small
!ls /content/so-small  # Verify that files are extracted
Initialize Spark and Load Data:
Create a cell that initializes the SparkSession and loads the Parquet files:

python
Copy
Edit
from pyspark.sql import SparkSession
spark = SparkSession.builder.appName("StackOverflow Analysis").getOrCreate()

posts_df = spark.read.parquet("/content/so-small/posts.parquet")
# Load other DataFrames as needed...
Run the Analysis Code:
Copy and paste the complete code (sections above) into a cell and run it step by step. Verify the outputs for schema, data exploration, aggregations, and plots.

View Visualizations and Results:
After running the analysis, the notebook will display the trend visualization and summary tables for fastest growing and declining languages.

Dependencies
Python 3.x

PySpark (should be available in Colab)

Matplotlib

Pandas

These dependencies are generally pre-installed or can be installed via pip if needed.

Notes
Data Filtering:
The date filter (2018-01-01) can be adjusted based on the available data and analysis period.

UDFs and Regular Expressions:
The UDF provided assumes tags are stored in a specific format (e.g., <python><java>). Adjust the regex if your tag format differs.

Spark Environment:
Running on Colab means you do not have direct access to your local file system; data must be uploaded or accessed via Google Drive/GCS.

License
This project is provided for educational purposes. You are free to use and modify the code as needed.

