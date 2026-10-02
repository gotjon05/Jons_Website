+++
date = '2026-10-02T12:42:04-05:00'
draft = false
title = 'Bulk Importing Constituent And Gift Information to Raisers Edge NXT'
+++

Goal: 

Provide a way for Constituent and Gift information to be imported into Blackbaud, while safely matching existing records 



Business Rules:

Existing constituents should be matched before creating a new constituent.
Gifts must be associated with the correct constituent before import.
Gifts need to be added to Blackbaud gift batch rather than posted directly 
A gift needed the required fields: Amount, Appeal, Package, Payment Method, Gift Date, Campaign, and Fund. 


Design:
My original program made the mistake of combining constituent matching with constituent update logic.Each condition checked for a single match using different criteria, but also contained its own copy of the steps for updating constituent information. This made the program hard to modify and maintain. 
I later refactored the program by creating child flows for individual operations, including constituent matching.This allowed the rest of the import process to use the same functions regardless of whether the constituent was matched or newly created.
The input for the import is loaded into a Sharepoint Excel Table, with Columns that included: ConstituentID, Email, BusinessPhone, MobilePhone, Attribute, Attribute_Description, Title, FirstName, LastName, company_name, company_position, business_address1, business_city, business_state, business_zip, Appeal, Package, SoftCreditTitle, SoftCreditName, SoftCreditLastName, Campaign, Fund, Amount, PaymentMethod, GiftDate, Addresstype, Addresslines, Addresscity, Addressstate, and Addresspostalcode.

I matched constituents using progressively broader criteria, that accepted matches when exactly one constituent was returned, otherwise checking broader criteria before creating a new record when there was no match.
1. Constituent ID
2. First Name + Last Name + Email
3. First Name + Last Name + Address + Postal Code
4. First Name + Last Name + Primary Business Name

If the matching child flow returned a System ID, I used that ID for the subsequent constituent update operations. If no match was returned, I created a new constituent and used the newly created system ID for the remaining operations.

The challenge in adding gifts to a gift batch was that the import table columns for Campaign, Appeal, Package and Fund are loaded with strings when the SKY API required the corresponding IDs for Campaign, Appeal, Package, and Fund when creating the gift.
After normalizing the text values, I filtered the corresponding SKY API records to find the matching Campaign, Appeal, Package, and Fund, then retrieved each record’s ID.I could then use those IDs, along with the remaining gift information from the import table, to add the gift to the batch I had created.

I also added the employer information using the fields: company_name, company_position, business_address1, business_city, business_state, business_zip. If the organization existed, I used its ID to create a relationship between the constituent and the organization, identifying the organization as the constituent’s primary employer. If no matching organization was found, I created a new organization record and then used its ID to create the employer relationship.


