+++
date = '2026-01-30T12:42:04-05:00'
draft = false
title = 'Automating Non-profit Acknowledgement Letters With Blackbaud API (Work in Progress)'
+++

This is a walkthrough for automating the creation of acknowledgement letters for Non-profits with Blackbaud NXT, using Blackbaud SKY API, Power Automate and Sharepoint.

**What this accomplishes:**

This workflow ensures that on a set schedule, every donor receives the right acknowledgment—addressed to the right person, at the right address, using the right letter template—without manual intervention.

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

  1. [To get Power Automate Premium $15/Month as of 2/14/26](https://go.microsoft.com/fwlink/?linkid=2252273&clcid=0x409&culture=en-us&country=us)

  2. [Go to Blackbaud Marketplace and search for power automate and Connect](https://app.blackbaud.com/marketplace/)

  3. Create a scheduled cloud flow in Power Automate

  4. Click the + sign and Search for Blackbaud Raisers Edge NXT List Gifts

  5. Blackbaud connector handles authentication internally and you will be prompted to sign-in with your blackbaud account 
  {{< /fold >}}

1. {{< fold title="Retrieving list of all Unacknowledged gifts" >}}
  Our first call is List_Gifts, to retrieve every unacknowledged gift. 
  The output provides a JSON array called value, which we loop through to access the information for each individual gift in the next step. 
  
  1. Click on List Gift and update the parameters:

  2. Start gift amount: 1 --- We want gifts that are greater than

  3. Acknowledgement status: NotAcknowledged

  4. Added on or after: (Optional, if your database is messy, you can select the earliest date for gifts you want. But you need a specific date format.) SELECT Fx and enter:   
```
  formatDateTime('2025-05-10', 'yyyy-MM-ddT00:00:00Z')
```
  1. Type: Enter the the gift types below to avoid Pledges: 
 ```
 Donation,GiftInKind,MatchingGiftPayment,PledgePayment,RecurringGiftPayment,Stock,SoldStock
 ```
   {{< img src="list_gifts.png" alt="List Gifts response in Power Automate" width="350" >}}
   {{< /fold >}}

1. {{< fold title="Looping through list of gifts" >}}
   We want to iterate over each gift returned by List_Gifts, so that we access all the information we need for every gift. To do this, we will use "apply to each" action. Most of the work after this will be inside this loop, where we make additional calls to gather more details about each gift.
  
  Before doing the steps below, run your process to see the output of List_Gifts. After it runs, go to Power Automates 28-day run history, select the most recent one, and select list_gifts and then select "show raw outputs". (This assumes you have unacknowledged gifts in Raisers Edge. If you don’t, create a few test gifts.)

  1. Select the + Sign after "List_Gifts" and Search for "apply to each"

  2. Add the action, then click Expression (fx) and enter: `@outputs('List_gifts')?['body/value']`
  Alternatively, you can select Dynamic content (lightning icon) to select body/value from List_Gifts. We want Body/value as the input, because its the section of the payload from List_Gifts that has the array of gifts we want to loop through.

   {{< img src="Apply_to_each.png" alt="apply to each" width="350" >}}

   {{< /fold >}}

4. {{< fold title="Retrieving gift information" >}}

   Our first call inside each iteration is Get a gift. We make this call to get gift date, gift amount, constituent ID, and appeal ID. Each time the flow runs, it makes this call once per gift.

   This call requires a gift ID, which comes from List Gifts:
   `items('Apply_to_each')?['id']`

  1. Expand "Apply to each" and select the + icon

  2. Search for blackbaud NXT get a gift

  3. Add the action, then enter the expression `items('Apply_to_each')?['id']` **or** select Dynamic content (lightning icon) and look for system record id of gift from List gifts

   {{< /fold >}}

5. {{< fold title="Storing Constituent ID from Get a Gift into a variable" >}}
  
  We reference **Constituent ID** multiple times throughout this flow. To avoid repeating expressions and to keep the flow readable, we store this value in a variable immediately after Get a Gift. 

  {{< fold title="How to initialize and set the variable" >}}

  1. Select the **+** icon below **Get a Gift**

  2. Search for **Initialize variable**

  3. Name the variable (e.g., `varConstituentId`) 

  4. Set the type to **String**  

  5. Set the value using the Constituent ID from **Get a Gift**

  {{< /fold >}}

   {{< /fold >}}

6. {{< fold title="Retrieving constituent information" >}}
   
  Our second call inside the iteration is Get Constituent. Like Get a gift, Each time the flow runs, it makes this call once per gift. We make this call to get Constituent Information about the person who received full/hard credit for the gift. (Where as Soft Credit is soft)

  1. Get a Constituent requires a Constituent ID. This is provided to us in Get a Gift. 

  2. select the + icon below, Get a Gift

  3. Search for blackbaud NXT Get a Constituent

  4. Add the action, then enter the expression 
  

   {{< /fold >}}

7. {{< fold title="Get Appeal and Package" >}}
  We retrieve the Appeal + Package for each gift because this combination determines which letter template we use.To do this safely, we add a Condition that checks whether an Appeal ID exists in each gift. If it does, we pull the appeal_id from Get a Gift


  {{< fold title="What is a Condition? And Why is it relevant?" >}}  
  A **Condition** in Power Automate is similar to an if/then statement. It evaluates an expression as True or False, often using AND or OR. 

  In this flow i have use **Conditions** to prevent running actions that depend on values that may be **null** (missing). Without these safeguards, Power Automate could attempt to call a connector with a blank ID (or reference missing data), which can cause the action—and sometimes the entire run—to fail.
  {{< /fold >}}

   {{< /fold >}}
8. {{< fold title="Resolving overlapping Appeal+Batch Letter Codes with Template Part 1" >}}
The Appeal + Batch Code from a gift determines which letter template to use.

But in this use case, the relationship was many-to-one. Many different Appeal + Batch combinations needed to map to the same letter template. For example, the “General” letter template had 5+ different Appeal + Batch combinations associated with it.

And this was a problem because later in the flow, I use a a single code to determine which template is generated for each gift, so Appeal + Batch alone wasn’t a reliable “single selector. 

In order to have a single code to indicate what template to use for each gift, I introduced LetterCodes that are 1-to-1 with templates. With each LetterCode having an array of Appeal+Batch codes that share the same LetterCode and template. 

To store this mapping, I created a JSON lookup table (an array of objects) where each object contains a LetterCode and an array of associated Appeal + Batch combinations.

And for each gift, i scan the appeal+batch code and returned the matching LetterCode. 

```json
  [
  {
    "LetterCode": "Gala",
    "AppealPackages": [
      "Gala - Gold",
      "Gala - Silver",
      "Gala - Bronze"
    ]
  },
  {
    "LetterCode": "General",
    "AppealPackages": [
      "General Donation",
      "Homepage Online",
      "Mailing End-of-Year",
      "Direct Mail Campaign"
    ]
  }
]
```
{{< /fold >}}


9. {{< fold title="Resolving overlapping Appeal+Batch Letter Codes with Template Part 2" >}}
{{< /fold >}}



10. {{< fold title="Identifying the type of Constituent to provide the correct Header and salutation later" >}}
{{< /fold >}}

11. {{< fold title="Creating Letters, as Adobe PDF's or Word Documents, and Marking Gifts as Acknowledged" >}}
{{< /fold >}}

12. {{< fold title="Emailing Acknowledgement Letters to Donors" >}}
{{< /fold >}}


{{< fold title="Review and Maintaining this Process" >}}
{{< /fold >}}
