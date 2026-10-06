+++
date = '2026-10-06T01:15:00-04:00'
draft = false
title = 'Building a Fundraiser Performance Dashboard in Power BI'
+++

Under Construction

The goal of this report was to measure fundraiser performance using data from Raiser’s Edge NXT. I focused the analysis on five areas: activity, pipeline progression, engagement with assigned constituents, fundraising outcomes, and follow-through. For three populations: Assigned Constituents, Assigned Constituents with Actions, and Assigned Constituents with Opportunities. I wanted to create a data model that would continuously update with new information. So I built an automated pipeline that retrieved constituent, fundraiser assignment, Action, Opportunity, gift, pledge, prospect status, wealth rating, and related fundraising data from Raiser’s Edge NXT through the SKY API. The API responses were transformed from JSON and array structures into structured datasets and written to SharePoint files, which served as refreshable data sources for a Power BI semantic model.

Requirements

- Build a data pipeline and model that could refresh as new data became available
- Drill down from Hospital -> Fundraiser Leader -> Fundraiser to evaluate performance at each level
- Analyze fundraiser performance over a rolling 12-month period
- Help fundraisers identify issues with their assigned prospects, actions and opportunities such as overdue Actions, stalled prospects, aging Opportunities, and gaps in portfolio engagement.
- Provide high-level performance summaries and detailed views of Assigned Constituents, Assigned Actions, and Assigned Opportunities.


Data Pipeline: 

I built an automated data pipeline using SKY API to continuously retrieve, transform and structure NXT data into separate tables for the Power BI model. Including the creation of fact, dimension and bridge tables,  


Fundraisers and Fundraiser Assignments

1. Ran List Fundraisers to retrieve active fundraisers from Raiser’s Edge NXT.
2. Appended each fundraiser into the DimFundraiser array with fields including Fundraiser Constituent ID, lookup ID, name, first name, and last name.
3. For each fundraiser, ran List Fundraiser Assignments to retrieve the constituents assigned to that fundraiser.
4. Used Select_FundraiserAssignments to extract the fundraiser ID, constituent ID, assignment ID, start date, assignment type, and amount.
5. Appended those assignment records into the DimFundraiserAssignments array.
6. Converted the fundraiser and assignment arrays into CSV tables to be used by model



Current Prospect Status

1. For each constituent returned through Fundraiser Assignments, ran Get Constituent Prospect Status using the assigned Constituent ID.
2. Checked whether a prospect status was returned before adding the record.
3. Used a Compose action to structure the returned fields: Constituent ID, Prospect Status, Days Elapsed, Status Start Date, Comments
4. Appended the structured record into the GetConstituentProspectStatus array.
5. After all assigned constituents had been processed, converted the array into a CSV table.
6. Updated the existing GetConstituentProspectStatus.csv file in SharePoint, which served as the refreshable Power BI data source.


Modelling and Bridge Tables: 

Before building the Power BI relationships, I created a modeling matrix that documented each business process, source table, grain, and the key fields used to relate the datasets. Defining the grain and relationships helped me understand the many-to-many relationships in the datasets, that could not be represented cleanly with direct relationships. 

FundraiserConstituentBridge: 

Why did I need a Fundraiser-Constituent Bridge Table: 

I needed to create a relationship between fundraisers and their assignment constituents. I could not place the fundraiser ID inside the constituent to create that same relationship without impacting the grain of one constituent per row, because a constituent could be assigned to multiple fundraisers.This also gave me a way to use the Fundraiser–Constituent Bridge to define which constituents belonged to the selected fundraiser’s portfolio. Because I could filter the bridge by the fundraiser table that pointed to it.

How I created the Bridge:

1. For each active fundraiser, I ran List Fundraiser Assignments to retrieve the constituents assigned to that fundraiser.
2. Used Select to extract the FundraiserConstituentID, DonorConstituentID, assignment ID, start date, assignment type, and amount.
3. Appended the selected assignment records into the DimFundraiserAssignments array, creating a nested array structure containing each fundraiser’s assignment records.
4. After all fundraiser assignments had been processed, converted DimFundraiserAssignments into a CSV table.
5. Used the CSV output to update the existing FundraiserAssignments.csv file in SharePoint.
6. Used that assignment dataset in Power BI to represent the Fundraiser–Constituent relationship, allowing a fundraiser to have many assigned constituents and a constituent to be associated with more than one fundraiser.























