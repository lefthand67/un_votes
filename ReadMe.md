# UN Votes Database

A comprehensive collection of normalized data on UN General Assembly resolution voting records from 1946 to the present, scraped from the UN Digital Library.

This database is designed as a resource for researchers and data scientists to conduct machine learning experiments with international relations data, such as building predictive models for future votes or clustering countries.

## Getting Started

For a detailed guide on how to install the database, understand its architecture, and run sample queries, please refer to the comprehensive tutorial:

👉 [**Tutorial.ipynb**](./Tutorial.ipynb)

## Project Details

- **Data Source:** [UN Digital Library](https://digitallibrary.un.org/search?c=Voting+Data&cc=Voting+Data&ln=en)
- **Update Frequency:** Updated monthly to include newly voted resolutions.
- **Key Features:** 12-table normalized PostgreSQL schema, covering resolutions, countries, vote choices, and agendas.

For current version statistics, see [RELEASE_NOTES.md](RELEASE_NOTES.md).
