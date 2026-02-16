+++
date = '2026-01-30T12:42:04-05:00'
draft = false
title = 'Automating Non-profit Acknowledgement Letters With Blackbaud API (Work in Progress)'
+++

This is a walkthrough for automating the creation of acknowledgement letters for Non-profits with Blackbaud NXT, using Blackbaud SKY API, Power Automate and Sharepoint.

**What this accomplishes:**\
This workflow ensures that every donor receives the right acknowledgment—addressed to the right person, at the right address, using the right letter template—without manual intervention.

{{< fold closed="The business rules that guided this process flow" >}}

- **Gifts considered in this Flow**  
  Only gifts greater than zero, not yet acknowledged, and not unpaid pledges.

- **Letter Template Matching Rule**  
  A gift’s **Appeal + Package** combination determines which letter template/content is used.

- **Business Identity Rule**  
  For individuals with business addresses, the flow identifies the correct Business Name using the constituent’s Primary Employer relationship.

- **Foundation Recipient Rule**  
  Foundation gifts without a soft credit, the flow identifies the correct recipient in this order:  
  *(Primary Contact → Principal/Director → Employee as catch-all)*

{{< fold closed="Distinct Letter Headers – 4 constituent scenarios" >}}

- **Individuals — Home Address**  
  - Title + First Name + Last Name  
  - Line 1  
  - City, State ZIP  

- **Individuals — Business Address**  
  - Title + First Name + Last Name  
  - Business Name  
  - Business Address  

- **Organization Gifts with Soft Credit**  
  - Title + Soft Credit First Name + Soft Credit Last Name  
  - Business Name  
  - Business Address  

- **Foundation Gifts**  
  - Title + First Name + Last Name of the selected contact  
  - Business Name  
  - Business Address  

{{< /fold >}}
{{< fold closed="Salutation Rules" >}}
  - Individuals → Dear Title Last Name,
  - Organizations with soft credit → Dear Soft Credit Title Last Name,
  - Foundations without soft credit → Dear Selected Contact Title Last Name,
{{< /fold >}}
{{< /fold >}}



1. {{< fold title="Getting Started: SKY API and Power Automate setup" >}}
  A. [To get access to the SKY API Libraries to create Create a SKY Developer account.](https://developer.blackbaud.com/skyapi/docs/getting-started) (You dont need to create an application)\
  B. [To get Power Automate Premium $15/Month as of 2/14/26](https://go.microsoft.com/fwlink/?linkid=2252273&clcid=0x409&culture=en-us&country=us)\
  C.






   {{< /fold >}}

2. {{< fold title="Retrieving list of all Unacknowledged gifts" >}}
  Our first call is List_Gifts, to retrieve every unacknowledged gift. 
  The output provides a JSON array called value, which we loop through to access the information for each individual gift. 
  
  A. Click the + sign and Search for Blackbaud Raisers Edge NXT List Gifts\
  B. Click on it and update the parameters:\
  C. Start gift amount: 1 --- We want gifts that are greater than\
  D. Acknowledgement status: NotAcknowledged\
  E. Added on or after: (Optional, if your database is messy, you can select the earliest date for gifts you want. But you need a specific date format.) SELECT Fx and enter:   
```
  formatDateTime('2025-05-10', 'yyyy-MM-ddT00:00:00Z')
```
 F. Type: Enter the the gift types below to avoid Pledges: 
 ```
 Donation,GiftInKind,MatchingGiftPayment,PledgePayment,RecurringGiftPayment,Stock,SoldStock
 ```



   {{< img src="list_gifts.png" alt="List Gifts response in Power Automate" width="350" >}}

   {{< /fold >}}

3. {{< fold title="Looping through list of gifts" >}}

   We want to iterate over each gift returned by List_Gifts, so that we access all the information we need for every gift. To do this, we will use "apply to each" action. Most of the work after this will be inside this loop, where we make additional calls to gather more details about each gift.
  
  Before doing the steps below, run your process to see the output of List_Gifts. After it runs, go to Power Automates 28-day run history, select the most recent one, and select list_gifts and then select "show raw outputs". (This assumes you have unacknowledged gifts in Raisers Edge. If you don’t, create a few test gifts.)

  A. Select the + Sign after "List_Gifts" and Search for "apply to each"\
  B. Add the action, then click Expression (fx) and enter: `@outputs('List_gifts')?['body/value']`\
  Alternatively, you can select Dynamic content (lightning icon) to select body/value from List_Gifts. We want Body/value as the input, because its the section of the payload from List_Gifts that has the array of gifts we want to loop through.

   {{< img src="Apply_to_each.png" alt="apply to each" width="350" >}}

   {{< /fold >}}

4. {{< fold title="Retrieving gift information" >}}

   Our first call inside each iteration is Get a gift. We make this call to get gift date, gift amount, constituent ID, and appeal ID. Each time the flow runs, it makes this call once per gift.

   This call requires a gift ID, which comes from List Gifts:
   `items('Apply_to_each')?['id']`

  A. Expand "Apply to each" and select the + icon\ 
  B. Search for blackbaud NXT get a gift\
  C. Add the action, then enter the expression `items('Apply_to_each')?['id']` **or** select Dynamic content (lightning icon) and look for system record id of gift from List gifts

   {{< /fold >}}

5. {{< fold title="Storing Constituent ID from Get a Gift into a variable" >}}
  
  We reference **Constituent ID** multiple times throughout this flow. To avoid repeating expressions and to keep the flow readable, we store this value in a variable immediately after Get a Gift. 

  {{< fold title="How to initialize and set the variable" >}}

  A. Select the **+** icon below **Get a Gift**  
  B. Search for **Initialize variable**  
  C. Name the variable (e.g., `varConstituentId`)  
  D. Set the type to **String**  
  E. Set the value using the Constituent ID from **Get a Gift**

  {{< /fold >}}

   {{< /fold >}}

6. {{< fold title="Retrieving constituent information" >}}
   
  Our second call inside the iteration is Get Constituent. Like Get a gift, Each time the flow runs, it makes this call once per gift. We make this call to get Constituent Information about the person who received full/hard credit for the gift. (Where as Soft Credit is soft)

  A. Get a Constituent requires a Constituent ID. This is provided to us in Get a Gift. 
  C. select the + icon below, Get a Gift
  D. Search for blackbaud NXT Get a Constituent
  E. Add the action, then enter the expression 
  

   {{< /fold >}}

7. {{< fold title="Get Appeal and Package" >}}
  We retrieve the Appeal + Package for each gift because this combination determines which letter template we use.To do this safely, we add a Condition that checks whether an Appeal ID exists in each gift. If it does, we pull the appeal_id from Get a Gift


  {{< fold title="What is a Condition? And Why is it relevant?" >}}  
  A **Condition** in Power Automate is similar to an if/then statement. It evaluates an expression as True or False, often using AND or OR. 

  In this flow i have use **Conditions** to prevent running actions that depend on values that may be **null** (missing). Without these safeguards, Power Automate could attempt to call a connector with a blank ID (or reference missing data), which can cause the action—and sometimes the entire run—to fail.


  {{< /fold >}}

   {{< /fold >}}
8. {{< fold title="Overlapping Appeal+Batch Letter Codes" >}}
Appeal+Batch informed what letter template to use for a gift. However, i encountered overlap where multiple Appeal+Batch corresponded to the same letter. To solve this problem, i created a LetterCode for each template and appeal+batch combinations that belonged to each letter. I did this by creating a JSON dictionary. My Key was the Letter Code and my value was the Appeal+Batch combinations. 



  {{< /fold >}}
9. {{< fold title="Identifying the type of Constituent to provide the correct Header and salutation later" >}}



  {{< /fold >}}