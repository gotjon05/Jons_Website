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


Creating Tables for Active Fundraisers and Fundraiser Assignments

1. Ran List Fundraisers to retrieve active fundraisers from Raiser’s Edge NXT.
2. Appended each fundraiser into the DimFundraiser array with fields including Fundraiser Constituent ID, lookup ID, name, first name, and last name.
3. For each fundraiser, ran List Fundraiser Assignments to retrieve the constituents assigned to that fundraiser.
4. Used Select_FundraiserAssignments to extract the fundraiser ID, constituent ID, assignment ID, start date, assignment type, and amount.
5. Appended those assignment records into the DimFundraiserAssignments array.
6. Converted the fundraiser and assignment arrays into CSV tables to be used by model



Creating Table for Current Prospect Status

1. For each constituent returned through Fundraiser Assignments, ran Get Constituent Prospect Status using the assigned Constituent ID.
2. Checked whether a prospect status was returned before adding the record.
3. Used a Compose action to structure the returned fields: Constituent ID, Prospect Status, Days Elapsed, Status Start Date, Comments
4. Appended the structured record into the GetConstituentProspectStatus array.
5. After all assigned constituents had been processed, converted the array into a CSV table.
6. Updated the existing GetConstituentProspectStatus.csv file in SharePoint, which served as the refreshable Power BI data source.


Creating Bridge Tables: 

Before building the Power BI relationships, I created a modeling matrix that documented each business process, source table, grain, and the key fields used to relate the datasets. Defining the grain and relationships helped me understand the many-to-many relationships in the datasets, that could not be represented cleanly with direct relationships. 


Creating FundraiserConstituentBridge Table: 

Why did I need a Fundraiser-Constituent Bridge Table: 

I needed to create a relationship between fundraisers and their assignment constituents. I could not place the fundraiser ID inside the constituent to create that same relationship without impacting the grain of one constituent per row, because a constituent could be assigned to multiple fundraisers.This also gave me a way to use the Fundraiser–Constituent Bridge to define which constituents belonged to the selected fundraiser’s portfolio. Because I could filter the bridge by the fundraiser table that pointed to it.

How I created the Bridge:

1. For each active fundraiser, I ran List Fundraiser Assignments to retrieve the constituents assigned to that fundraiser.
2. Used Select to extract the FundraiserConstituentID, DonorConstituentID, assignment ID, start date, assignment type, and amount.
3. Appended the selected assignment records into the DimFundraiserAssignments array, creating a nested array structure containing each fundraiser’s assignment records.
4. After all fundraiser assignments had been processed, converted DimFundraiserAssignments into a CSV table.
5. Used the CSV output to update the existing FundraiserAssignments.csv file in SharePoint.
6. Used that assignment dataset in Power BI to represent the Fundraiser–Constituent relationship, allowing a fundraiser to have many assigned constituents and a constituent to be associated with more than one fundraiser.

Creating ActionFundraiserBridge Table:  

Why did I need an Action Fundraiser Bridge? I needed to create a relationship between fundraisers and their actions. 
An action can be associated with more than one fundraiser. If I tried to store a single FundraiserID directly in the Action table to represent a relationship between FundraiserID and ActionID, the duplicate relationships between different fundraisers and an action would break the grain of one Actionid per row. Which I wanted to keep, to avoid distorting counts, averages, percentages, later in the process when creating measures.
The Actions Table already had constituent IDs for each ActionID. My main concern was the ability to filter for fundraisers. With Actions and Fundraisers pointed at my Action Fundraiser bridge table, I was able to filter using actionfundraiser bridge table for actions that were associated with that fundraiser

How I created the Bridge:
1. For each fundraiser and each of their assigned constituents, ran List Constituent Actions.
2. For each returned Action, extracted the ActionID and the Action’s associated fundraisers collection.
3. Appended those results into the ActionBridgeFundraiser array, creating a nested array structure where each Action contained its associated fundraiser collection.
4. After all fundraiser assignments and Actions had been processed, converted the ActionBridgeFundraiser variable into a CSV table.
5. Used that CSV output to update the existing ActionFundraiserBridge.csv file in SharePoint.
6. That SharePoint CSV served as the refreshable source for the Action–Fundraiser relationship in the Power BI semantic model.


OpportunityFundraiserBridge

The same decisions for actions applied to Opportunities. 

How I created the Bridge:

1. Used the records returned from List Opportunities.
2. For each returned Opportunity, extracted the OpportunityID and the Opportunity’s associated fundraisers collection.
3. Appended those results into the OpportunityBridgeFundraiser array, creating a nested array structure where each Opportunity contained its associated fundraiser collection.
4. After all fundraisers and Opportunities had been processed, converted the OpportunityBridgeFundraiser variable into a CSV table.
5. Used the CSV output to update the existing OpportunityFundraiserBridgeTable.csv file in SharePoint.
6. That SharePoint CSV served as the refreshable source for the Opportunity–Fundraiser relationship in the Power BI semantic model.









Organizational hierarchy / drill-down: 

Each Matrix needed a drill-down by Market (Hospital), Fundraiser Leader, Fundraiser to analyze fundraiser performance at different levels of the organization
I worked with senior leadership to get an organizational mapping between each fundraiser and their fundraiser leader and merged that information into Fundraiser table using the Fundraiser Constituent ID
I created a separate Market/Network lookup table that mapped each MarketID to its Market and Network. Actions and Opportunities contained MarketID, allowing them to relate to this table and be drilled down by market and network


Report:

I broke up the report into three segments focused on different populations. Assigned Constituents, Assigned Constituents with Actions and Assigned Constituents with Opportunities. Each segment had its own matrix and measures designed to evaluate different aspects of fundraiser performance, including activity, pipeline progression, engagement with assigned constituents, fundraising outcomes, and follow-through. 

Criteria for evaluating performance 

Activity = How much work is the fundraiser doing, and when?
number of actions, completed actions, overdue actions, recent activity, and the distribution of activity across your rolling 18-month window.
Engagement with assigned constituents = How much of the fundraiser’s assigned portfolio is actually being touched or developed? what percentage of assigned constituents had Actions, Opportunities, Gifts, multiple completed Actions, recent contact, etc.
Follow-through = whether they are completing actions and keeping work from becoming overdue or stagnant 
Results = what that work ultimately produced 


Fundraiser Overview — assigned constituents, constituents with Actions %, constituents with Opportunities %, constituents with Gifts %, event participation/engagement, Action completion, Action follow-through, active/open Opportunities, pipeline value, Opportunity outcomes. 
Assigned Constituents — portfolio coverage and engagement: assigned count, constituents with Actions, high-value Actions, completed Actions, Opportunities, Gifts, pledges, event attendance, average years giving.
Assigned Actions — fundraiser activity and follow-through: total Actions, high-value Actions %, completed Actions %, past due Actions %, open Actions %, total open or past due Actions, days since last Action, average completion time, recent activity, activity distribution over time.
Assigned Opportunities — pipeline progression and fundraising outcomes: accepted Opportunities, value of closed-accepted Opportunities, closed Opportunities accepted %, value of open Opportunities, open Opportunity count, pledge Opportunity count, overdue Opportunities %, average age of open Opportunities, pipeline composition, ask amounts, funded amounts.


Dashboards

1. Stalled Prospect Statuses: The goal was to give fundraisers a clear view of their assigned constituents who had stalled in the prospect status pipeline based on the number of days elapsed. Each prospect status had different green, yellow and red elapsed time thresholds to indicate urgency.
1. I retrieved the data from GetConstituentProspectStatus
3. Created a relationship with constituent table based on const id
4. Created DAX measures using SWITCH(TRUE()) to capture the different elapsed-day ranges for each prospect status and return 1, 2, or 3. I couldn’t use one simple set of conditional-formatting rules directly on the elapsed-days value because each prospect status had different thresholds for what counted as green, yellow, or red.
5. Used those values to define the conditional-formatting rules
6. Activity-distribution matrix across the rolling 18-month window in 3-month buckets: The goal was to compare and identify periods of concentrated or declining fundraiser activity across a rolling 18-month period by grouping their distinct action activity into 3-month buckets and comparing the distribution of actions over time. 
7. Each column in the matrix represented a 3 month period within the rolling 18-month window. Calculating Distinct Actions in the 3-month bucket ÷ Total Distinct Actions for that fundraiser across the full 18-month window
This returned the percentage of each fundraiser’s total activity that occurred during each 3-month period, making increases, declines, and concentrations of activity easier to identify.

Related: weekly prospect screening / prospect quality dashboard: The goal was to provide a high level snapshot of Grateful prospect donors of interest to fundraisers, each week.
















