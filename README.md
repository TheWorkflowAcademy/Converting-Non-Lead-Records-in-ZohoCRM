# ZohoCRM-Convert-Record-Across-Modules
A Deluge script that replicates the conversion of a record across different modules in Zoho CRM by copying info to a new record in a different module and deleting the old record.

## Core Idea
Zoho CRM has a default conversion button that lets users convert Leads to Contacts on a click of a button. This functionality however, is limited to only the 2 aforementioned modules. This script aims to replicate the conversion button should you need to convert records across modules that aren't Leads to Contacts. This is done by copying all the necessary fields and related lists from the original module to the new, then deleting the old record.

## Example Case
You are running an insurance business where your client accounts are a combination of both Households and Businesses. You manage these 2 different types of accounts in CRM by having 2 modules - Households & Businesses. At any point of time, your staff needs to convert Household records to Businesses and vice versa.
