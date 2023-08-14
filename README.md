# PIANO-# Piano.io Data Integration with Google Cloud Platform

Welcome to the documentation for the Piano.io Data Integration project. This project focuses on seamlessly connecting Piano's highly customizable API data with the Google Cloud Platform (GCP). 
By following this guide, you'll learn how to utilize Piano API documentation, with GCP Service Account credentials, using Python programming, and data warehousing techniques to efficiently manage and analyze your Piano.io data.

## Table of Contents

- [Introduction](#Introduction)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
- [Project Structure and Overview](#project-structure)
- [Installation](#installation)
- [Usage](#usage)
- [Webhooks Integration](#webhooks-integration)
- [Data Analysis](#data-analysis)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

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

## Simple Usage

Creating firs connection (Heand shake): 
*Note that there is a limit I've added for a test.

```
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
```
self.call_Urls = {
                     'all_platform_conversions': 'https://sandbox.piano.io/api/v3/publisher/conversion/list',
                     'all_users_accesses': 'https://sandbox.piano.io/api/v3/publisher/user/list/accesses',
                     'all_users': 'https://sandbox.piano.io/api/v3/publisher/user/list',
                     'all_terms': 'https://sandbox.piano.io/api/v3/publisher/term/list',
                     "list_resource": 'https://sandbox.piano.io/api/v3/publisher/resource/list',
                     'list_subscriptions': 'https://sandbox.piano.io/api/v3/publisher/subscription/list',
                     'list_promotions': 'https://sandbox.piano.io/api/v3/publisher/promotion/list',
                     'list_offers': 'https://sandbox.piano.io/api/v3/publisher/offer/list',
                     'list_all_webhooks': 'https://sandbox.piano.io/api/v3/publisher/webhook/list'
                      #.... Add any endpoint based on the provided design document

                     }
```

The following are several examples of api calls based on the provided design document:
Include them into the primary class. 
Any bloc API data call using the above endpoint urls are starting with:
```
all_users = requests.post(self.call_Urls["all_users"],
                                  params=self.body, headers=self.headers)

users = all_users.json()

```
After calling the API endpoint, you'll get a json with the data.
As I've mentioned, my goal would be to create a data warehouse in Google BigQuery using snowflake schema.
In order to achieve that Its important to keep in mind the general joins as well as micro-joins, dim and fact tables.

Another important notion is that you'll have to consider both the design document provided by the department asking for the data in its de normilized form or even a graph but aloso, as a data developer, I had to consider the other data warehouse in my data lake that I will have to use in order to get the appropreate result. i.e, to determin ont only joins within (Intra) the Piano.io json results (in tabular form) but inter-DWH sometimes cross-projects DWH. 

In the bellow example, I'm adding a method to my `GetPianoData` Class based on the above points.
```
def pianoGetUsers(self):
        all_users = requests.post(self.call_Urls["all_users"],
                                  params=self.body, headers=self.headers)

        users = all_users.json()
        users_details = {item['uid']: [item['first_name'], item['last_name'],
                                       item['create_date'], item['email']] for item in users['users']}

        userDF = pd.DataFrame(users_details).T.reset_index()
        userDF.columns = ['uid', 'firstName', 'lastName', 'create_date', 'email']

        userDF['create_date'] = pd.to_datetime(userDF['create_date'], unit='s') #I've de

        return userDF
```
## Piano.io -json 'if' exsis

In Piano.Io API Json results are tricky. rach endpoint call reflects all key objects and attributes, including elements that are present in any other api endpoint call. 
Therefore, each different call with defferent data will consiss repeated objects in the json. 

I had to go throught the jsons and 'fish' the data relevant based on the outlook of my data warehouse building strategy.
When it comes to data in the json that reflects sometimes repeated objects, but sometimes - simply missing important objects or elements, my solution was to write the following:

```
    def pianoGetTerms(self):

        terms_data = requests.post(self.call_Urls["all_terms"],
                                   params=self.body, headers=self.headers)

        all_terms = terms_data.json()

        my_data = {item['term_id']: [
            item['resource']['rid'],
            item['resource']['name'],
            item['resource']['resource_url'],
            item['name'],
            item['description'],
            item['type'],
            item['type_name'],
            item['create_date'],
            item['update_date'],
            item['collect_address'],
            item['external_api_id'] if 'external_api_id' in item else "-",
            item['external_api_name'] if 'external_api_name' in item else "-",
            item['payment_currency'] if 'payment_currency' in item else "-",
            item['payment_billing_plan'] if 'payment_billing_plan' in item else "-",
            item['payment_billing_plan_description'] if 'payment_billing_plan_description' in item else "-",
            item['payment_currency'] if 'payment_currency' in item else "-",
            item['external_api_form_fields'][0]['field_name'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['field_title'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['description'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['mandatory'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['hidden'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['default_value'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['order'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['type'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-",
            item['external_api_form_fields'][0]['editable'] if 'external_api_form_fields' in item and item[
                'external_api_form_fields'] else "-"
        ] for item in all_terms['terms']
        }
        terms_dataDF = pd.DataFrame(my_data).T.reset_index()
        terms_dataDF.columns = ['term_id', 'resourceId', 'resource_name', 'resource_url',
                                'term_name', 'description', 'type', 'type_name', 'create_date', 'update_date',
                                'collect_address', 'external_api_id', 'external_api_name', 'payment_currency',
                                'payment_billing_plan', 'payment_billing_plan_description', 'payment_currency',
                                'external_api_form_fields_name', 'external_api_form_fields_title',
                                'external_api_form_fields_description',
                                'external_api_form_fields_mandatory', 'external_api_form_fields_hidden',
                                'external_api_form_fields_default_value', 'external_api_form_fields_order',
                                'external_api_form_fields_type', 'external_api_form_fields_editable'
                                ]

        terms_dataDF['create_date'] = pd.to_datetime(terms_dataDF['create_date'], unit='s')
        terms_dataDF['update_date'] = pd.to_datetime(terms_dataDF['update_date'], unit='s')

        return terms_dataDF
```

## Webhooks Integration

This project also supports the integration of Piano.io Webhooks data. Follow the steps in the `webhooks/` directory to set up Webhooks to automatically capture new subscriber data.
Based on the provided design document I've got all the webhooks that has been set in the main project however, if the DD states that they need only certain subscriber's actions captures, then it is up to you which to filter out.

```
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

##


## Data Analysis

Explore the Jupyter notebooks in the `notebooks/` directory to gain insights from your integrated data. Visualize trends, subscriber growth, and other valuable metrics.

## Contributing

We welcome contributions from the community! If you have any improvements or suggestions, feel free to open issues or submit pull requests.

## License

This project is licensed under the [MIT License](LICENSE).

## Contact

For any questions or feedback, please contact [your email address].

Happy data integrating!
