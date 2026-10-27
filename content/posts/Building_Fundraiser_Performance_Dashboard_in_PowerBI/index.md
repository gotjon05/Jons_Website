+++
date = '2026-10-06T01:15:00-04:00'
draft = false
title = 'My experience creating a Fundraiser Performance Report in Power BI'
+++

Under Construction


Overview: 

I want this write-up to summarize and reflect on a report I created for a hospital's fundraising department, to evaluate fundraiser performance using data collected from Blackbaud NXT. 

Fundraiser Leadership wanted a high level overview of fundraiser performance and a way to use available data to better support their fundraisers. I was given a broad scope with the intention that the requirements would be more defined through stakeholder feedback.

My goal for this report was to create a report that could continue to be relied upon even if i was no longer there, with a data model that was continuously fed new information.
   
Requirements that evolved over time: 

- Build a data pipeline and model that could refresh as new data became available
- Drill down from Hospital -> Fundraiser Leader -> Fundraiser to evaluate performance at each level
- Analyze fundraiser performance over a rolling 12-month period
- Help fundraisers identify issues with their assigned prospects, actions and opportunities such as overdue Actions, stalled prospects, aging Opportunities, and gaps in portfolio engagement.
- Provide high-level performance summaries and detailed views of Assigned Constituents, Assigned Actions, and Assigned Opportunities.


The Process in creating a Date Pipeline: 

Fundraisers log interactions and solicitations in their assigned constituent records. This was the primary information i extracted and used for my report. I started with the expectation that their would be unique one to many relationships between fundraisers and the Constituent, Actions and Opportunities they were assigned to. I began extracting active fundraisers and their assigned constituents, without considering that constituents could have more than one fundraiser.

While examining the Action data returned from SKY API, I discovered that the fundraisers array could contain multiple fundraisers for the same action. Starting the conversation that led to me recognizing that fundraisers and their assigned constituents, actions and opportunities could have many to many relationships.

Action and Opportunity endpoints had a grain of one distinct action/opportunity per row, but each with an array for multiple fundraisers. Fundraiser Assignments had duplicate constituents for different fundraisers.    

I wanted to create a relationship between each individual fundraiser and each action, opportunity and constituent while preserving the grain of one row per unique record. 

One option was to expand the fundraiser array for Opportunity and Action, creating a separate row for each distinct Action-Fundraiser or Opportunity-Fundraiser. But that would have changed the grain by creating duplicate rows and increase the risk of counting duplicate values in aggregations.  

The alternative was to create bridge tables to have 1 to many relationships between fundraisers, and their constituents, actions and opportunities. Presented by KimballGroup.com I used that as a guide to create bridge tables. 

I created bridge tables with distinct pairings of action/opportunity/constituent with the fundraiser ID. Allowing me to create a relationship where the fundraiser table was 1:many with the bridge. and the action/opportunity/constituent table was 1 to many with the action/opportunity/constituent. 



{{< fold title="Creating the Bridge Tables" >}}
Extracting data for Fundraiser-Constituent Bridge Table With Distinct Pairings of Active Fundraiser and Assigned Constituent:

1. For each active fundraiser, I ran List Fundraiser Assignments to retrieve the constituents assigned to that fundraiser.
2. Used Select to extract the FundraiserConstituentID, DonorConstituentID, assignment ID, start date, assignment type, and amount.
3. Appended the selected assignment records into the DimFundraiserAssignments array, creating a nested array structure containing each fundraiser’s assignment records.
4. After all fundraiser assignments had been processed, converted DimFundraiserAssignments into a CSV table.
5. Used the CSV output to update the existing FundraiserAssignments.csv file in SharePoint.
6. Used that assignment dataset in Power BI to represent the Fundraiser–Constituent relationship, allowing a fundraiser to have many assigned constituents and a constituent to be associated with more than one fundraiser.


Extracting data for Fundraiser-Action Bridge Table with Distinct Pairings of Active Fundraisers And Actions:

1. For each fundraiser and each of their assigned constituents, ran List Constituent Actions.
2. For each returned Action, extracted the ActionID and the Action’s associated fundraisers collection.
3. Appended those results into the ActionBridgeFundraiser array, creating a nested array structure where each Action contained its associated fundraiser collection.
4. After all fundraiser assignments and Actions had been processed, converted the ActionBridgeFundraiser variable into a CSV table.
5. Used that CSV output to update the existing ActionFundraiserBridge.csv file in SharePoint.
6. That SharePoint CSV served as the refreshable source for the Action–Fundraiser relationship in the Power BI semantic model.

Extracting data for Fundraiser-Opportunity Bridge Table with Distinct Pairings of Active Fundraisers and Opportunities:

1. Used the records returned from List Opportunities.
2. For each returned Opportunity, extracted the OpportunityID and the Opportunity’s associated fundraisers collection.
3. Appended those results into the OpportunityBridgeFundraiser array, creating a nested array structure where each Opportunity contained its associated fundraiser collection.
4. After all fundraisers and Opportunities had been processed, converted the OpportunityBridgeFundraiser variable into a CSV table.
5. Used the CSV output to update the existing OpportunityFundraiserBridgeTable.csv file in SharePoint.
6. That SharePoint CSV served as the refreshable source for the Opportunity–Fundraiser relationship in the Power BI semantic model.

{{< /fold >}}





























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


Creating FundraiserConstituentBridge Table: 

Why did I need a Fundraiser-Constituent Bridge Table: 


Creating ActionFundraiserBridge Table:  

Why did I need an Action Fundraiser Bridge? I needed to create a relationship between fundraisers and their actions. 
An action can be associated with more than one fundraiser. If I tried to store a single FundraiserID directly in the Action table to represent a relationship between FundraiserID and ActionID, the duplicate relationships between different fundraisers and an action would break the grain of one Actionid per row. Which I wanted to keep, to avoid distorting counts, averages, percentages, later in the process when creating measures.
The Actions Table already had constituent IDs for each ActionID. My main concern was the ability to filter for fundraisers. With Actions and Fundraisers pointed at my Action Fundraiser bridge table, I was able to filter using actionfundraiser bridge table for actions that were associated with that fundraiser











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
















