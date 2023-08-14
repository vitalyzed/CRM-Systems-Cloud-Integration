# CRM Cloud Integration Project-# Piano.io Data Integration with Google Cloud Platform

Welcome to the documentation for the Piano.io Data Integration project. This project focuses on seamlessly connecting Piano's highly customizable API data with the Google Cloud Platform (GCP). 
By following this guide, you'll learn how to utilize Piano API documentation, with GCP Service Account credentials, using Python programming, and data warehousing techniques to efficiently manage and analyze your Piano.io data.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Project Structure and Overview](#project-structure-and-overview)
  -[Define class](#define-class)
- [Installation](#installation)
- [Class structure Calls](#class-structure-calls)
- [Webhooks Integration](#webhooks-integration)
- [Contributing](#contributing)

## Introduction

The Piano.io Data Integration project aims to enable seamless extraction and utilization of Piano.io data using API into Google Cloud Platform (GCP).
By establishing a connection between Piano's API and GCP's data storage and analysis tools, this project streamlines data processing and empowers data-driven decision-making.

## Prerequisites 

Before you begin, make sure you have the following prerequisites: 

- Piano.io API access and API documentation https://docs.piano.io/api/
- Google Cloud Platform account with Service Account credentials for relevant services.
- Proficiency in Python programming for data manipulation and integration.
- Basic knowledge of data warehousing and data lakes.
- Understanding of JSON data structure.

## Getting Started

To get started with this project:

1. Set up your IDE (I'll be using Pycharm IDE), using Python 3.11
2. Set up your GCP Service Account credentials and ensure the necessary permissions
  ***Please note: Best Practices in IAM when it comes to GCP are to avoid giving your service account "Owner" role.
4. Review the Piano.io API documentation to understand available endpoints and data structures.
   *https://docs.piano.io/

## Project Structure and Overview
Breaking the script into main directories:
1. `api_classes/`: contain the script that calls all the data needed based on a given design document (contains several examples)
2. `prep_for_cloud_upload/`: contains the script that preparing the data for GCP storage upload. Best practices and tips included.
3. `source_subs/`: contains the script that uses Piano.io API for extracting ready reports via report ID
4. `uploading_content`: contains the script that is responsible to upload the data retrieved from Piano.io to GCP

The project is organized as follow

- `src/`: Contains Python scripts for data extraction, transformation, and loading.
- `notebooks/`: Jupyter notebooks showcasing data analysis and visualization.
- `data/`: Placeholder for data files retrieved from Piano.io and stored locally.
- `config/`: Store configuration files for API keys, GCP credentials, etc.

## Installation

1. Install the required Python packages listed in `requirements.txt` using `pip`.
2. Configure your API keys and GCP Service Account credentials in the `config/` directory.

## Class structure Calls

Creating firs connection (Heand shake): 
*Note that there is a limit I've added for a test.

##### Define class
```python
class GetPianoData:
    def __init__(self, api_key, aid):

        self.api_key = api_key 
        self.aid = aid

        self.headers = {
            "Content-Type": "application/x-www-form-urlencoded",
            "Accept": "application/json"
        }

        self.body = {"aid": aid,
                "offset": 0,
                "api_token": api_key,
                "include_disabled": 'true',
                "limit": 500
                }
```

Add endpoint urls for the call **Please note that I'm using the Sandbox version (not production)**
```python
self.call_Urls = {
                     'all_platform_conversions': 'https://sandbox.piano.io/api/v3/publisher/conversion/list',
                     'all_users_accesses': 'https://sandbox.piano.io/api/v3/publisher/user/list/accesses',
                      #.... Add any endpoint based on the provided design document

                     }
```

The following are several examples of api calls based on the provided design document:
Include them into the primary class. 
Any bloc API data call using the above endpoint urls are starting with:
```python
all_users = requests.post(self.call_Urls["all_users"],
                                  params=self.body, headers=self.headers)

users = all_users.json()

```
After calling the API endpoint, you'll get a json with the data.
As I've mentioned, my goal would be to create a data warehouse in Google BigQuery using snowflake schema.
In order to achieve that Its important to keep in mind the general joins as well as micro-joins, dim and fact tables.

Another important notion is that you'll have to consider both the design document provided by the department asking for the data in its de normilized form or even a graph but aloso, as a data developer, I had to consider the other data warehouse in my data lake that I will have to use in order to get the appropreate result. i.e, to determin ont only joins within (Intra) the Piano.io json results (in tabular form) but inter-DWH sometimes cross-projects DWH. 

In the bellow example, I'm adding a method to my `GetPianoData` Class based on the above points.
```python
def pianoGetUsers(self):
        all_users = requests.post(self.call_Urls["all_users"],
                                  params=self.body, headers=self.headers)

        users = all_users.json()
        users_details = {item['uid']: [item['first_name'], item['last_name'],
                                       item['create_date'], item['email']] for item in users['users']}

        userDF = pd.DataFrame(users_details).T.reset_index()
        userDF.columns = ['uid', 'firstName', 'lastName', 'create_date', 'email']

        userDF['create_date'] = pd.to_datetime(userDF['create_date'], unit='s') 

        return userDF
```
## Piano.io -json 'if' exsists

In Piano.Io API Json results are tricky. rach endpoint call reflects all key objects and attributes, including elements that are present in any other api endpoint call. 
Therefore, each different call with defferent data will consiss repeated objects in the json. 

I had to go throught the jsons and 'fish' the data relevant based on the outlook of my data warehouse building strategy.
When it comes to data in the json that reflects sometimes repeated objects, but sometimes - simply missing important objects or elements, my solution was to write the following:

using the following example, calling the Piano.io Term object API endpoint:
```python
    def pianoGetTerms(self):

        terms_data = requests.post(self.call_Urls["all_terms"],
                                   params=self.body, headers=self.headers)

        all_terms = terms_data.json()
```
Then starting to "fish" for the appropreate data based on the DD given and the Data general model structure:

```python

        my_data = {item['term_id']: [
            item['resource']['rid'],
            ......
            ......
            ......
            item['external_api_id'] if 'external_api_id' in item else "-",
            item['external_api_name'] if 'external_api_name' in item else "-",
            ......
            ......
            item['external_api_form_fields'][0]['field_name'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['field_title'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            ......
            ......
            ......
        ] for item in all_terms['terms']
        }
        terms_dataDF = pd.DataFrame(my_data).T.reset_index()
        terms_dataDF.columns = ['term_id', 'resourceId', 'resource_name', 'resource_url',
                                            ......
                                            ......
                                            ......

                                ]

        terms_dataDF['create_date'] = pd.to_datetime(terms_dataDF['create_date'], unit='s')
        terms_dataDF['update_date'] = pd.to_datetime(terms_dataDF['update_date'], unit='s')

        return terms_dataDF
```

## Webhooks Integration

This project also supports the integration of Piano.io Webhooks data. Follow the steps in the `webhooks/` directory to set up Webhooks to automatically capture new subscriber data.
Based on the provided design document I've got all the webhooks that has been set in the main project however, if the DD states that they need only certain subscriber's actions captures, then it is up to you which to filter out.

```python
 def webhooks_all(self):
        all_events = requests.post(self.call_Urls["list_all_webhooks"],
                                   params=self.body, headers=self.headers)

        webhooks_all = all_events.json()
        webhooks_details = {item['webhook_id']: [item['status'],
                                                 item['retried'],
                                                 item['type'],
                                                 item['create_date'],
                                                 item['update_date'] if 'update_date' in item else "-",
                                                 item['event_localized'],
                                                 item['user']['uid']
                                                 ] for item in webhooks_all['webhooks']
                            }
        webhooksDF = pd.DataFrame(webhooks_details).T.reset_index()
        webhooksDF.columns = ['webhook_id', 'status', 'retried', 'type', 'create_date',
                              'update_date', 'event_localized', 'user']

        webhooksDF['create_date'] = pd.to_datetime(webhooksDF['create_date'], unit='s')
        webhooksDF['create_date'] = webhooksDF['create_date'].dt.strftime('%Y-%m-%d')

        return webhooksDF
```

## Preparing to upload

This part contains the script that prepares the data to upload to the Cloud. 
Based on the Design document and with coherentness with my building model and strategy regarding data modeling and database structures, I'm uploading to GCP storage.

A tip: 
Best for debugging, its consedered best practice to use a blue print of my previous codes in order to make the debugging proccess familiar, commun and easy to get.
Not only for me, but also for the other developers in my team. More about that in later section.

```python
import json
from google.cloud import bigquery
from google.cloud import storage
```

By using this blue-print, I've managed to do the following:
- maintain the script for versions easily as each part is repreated.
- debug easely due to familiarity.
- add new data using the Class OOP the rest is a blue-print copy.

```python
def write_dataframe_to_bucket(bucket_name, file_name, dataframe):
    csv_data = dataframe.to_csv(index=False)
    storage_client = storage.Client()
    bucket = storage_client.bucket(bucket_name)
    blob = bucket.blob(file_name)
    blob.upload_from_string(csv_data, content_type='text/csv')
```

Then:
```python
def load_table(bigquery_client,uri,table_id):

    job_config = bigquery.LoadJobConfig(
        autodetect=True,
        max_bad_records=2000,
        skip_leading_rows=1,
        source_format=bigquery.SourceFormat.CSV,
    )
    load_job = bigquery_client.load_table_from_uri(
        uri, table_id, job_config=job_config
    )
    load_job.result()
    destination_table = bigquery_client.get_table(table_id)
    print(f"Report loaded {format(destination_table.num_rows)} rows to table: {table_id}.")
```
Assuming You are going to execute this cloud function during the night based on the previous day for the analysts to add to their dashboards:

```python
def get_yesterdays():
    from datetime import date, timedelta
    yesterday = date.today() - timedelta(days=1)
    return yesterday.strftime('%Y%m%d')

def prepare_piano_files(file_name,data):
    bucket = "PROJECT_NAME"
    file = f'path/to_my_file/in_bucket/{file_name}.csv'

    write_dataframe_to_bucket(bucket, file, data)
    table_id = f'project_id.table_id.{file_name}' 

    bigquery_client = bigquery.Client()
    uri = f'gs://path/to_my_file/in_bucket/{file_name}.csv'

    load_table(bigquery_client, uri, table_id)

def update_Piano_files(piano_file_name,data):
    file_name=piano_file_name
    prepare_piano_files(file_name,data)
```

And Finaly, in the --- directory:
```python
from piano_api_mod import GetPianoData,api_key,aid
from PianoGCP_func import prepare_piano_files,update_Piano_files,get_yesterdays
from piano_articles_source import get_articles_subs_report
import os

os.environ["GOOGLE_APPLICATION_CREDENTIALS"] = "your_GCP_dervice_account.json"

def update_Piano_BQ():
    myData = GetPianoData(api_key, aid)


    #Updating Promotions:
    promotionData = myData.pianoGetPromotions()
    piano_file_name='sb_Promotions_'+get_yesterdays()
    update_Piano_files(piano_file_name,promotionData)

...
...
...

```

## Contributing

We welcome contributions from the community! If you have any improvements or suggestions, feel free to open issues or submit pull requests.

