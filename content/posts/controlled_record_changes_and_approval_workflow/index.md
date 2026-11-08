+++
date = '2026-11-07T12:42:04-05:00'
draft = false
title = 'Controlled Record Changes and Approval Workflow (Work in Progress)'
+++


Users were requesting access to modify CRM records. But my boss wanted to limit access to maintain data quality. 

To balance both needs, I created a staging and approval workflow where users could request changes to a record. And once approved, those changes automatically took effect. 

User would visit a record, and use a form to submit updates for constituent, address, phone and email to make the changes they wanted. That information then went to a Sharepoint List where staff could view the suggested changes, and select "Approve". Once Approved,  
another program would check the list for Approved, and create or update the changes in NXT. The requester would then be notified via email that the changes were made. 

The flow provided three primary functions: 

1. Validated users identity token through Blackbaud SKY Add-in Connector and generated Adaptive Card containing fields: Name, DOB, Marital Status, Email, Phone and Mailing Address

2. Retrieve the response when the user selects Submit in the adaptive card, and store it in Sharepoint List with the status = Pending 

3. Check Sharepoint List for records with a status = "Approved" and Processed = "No". Then updated or created the requested updates, and marked the request as processed and notified the requester of completion via email. 



Flow 1: Adaptive Card Creation

I registered a SKY application with Constituent Tile Dashboard extension point in Blackbaud's developer portal. When a user visited a constituent record and the tile loaded, this application would call my first flow to generate the form. 

The first flow had a trigger of "When an HTTP request is received". And the SKY application would send an HTTP request to my flow.
 
My Flow responded with a generated the Adaptive Card JSON, and returned it through an HTTP Response action.


Flow 2: Receiving data from user 


