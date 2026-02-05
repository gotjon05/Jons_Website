+++
date = '2026-01-30T12:42:04-05:00'
draft = false
title = 'Automating Non-profit Acknowledgement Letters With Blackbaud API (Work in Progress)'
+++

This is a walkthrough for automating the creation of acknowledgement letters for Non-profits with Blackbaud NXT.

What this accomplishes:
- Retrieves all unacknowledged gifts from Raiser’s Edge NXT (SKY API) and pulls the related gift + constituent details needed for letter personalization.
- Uses an Appeal/Package to LetterCode mapping to select the correct letter template/content.
- Builds accurate headers and salutations for individuals vs organizations/foundations, including preferred business addresses and primary contacts when required.
- Generates the acknowledgment letter from a template, saves the output, emails the donor, and then updates the gift acknowledgment status


{{< fold closed="The business rules that guided this process flow" >}}

- **Appeal + Package Rule**  
  Gifts Appeal + Package combination determines which letter template/content is used.

- **Business Identity Rule**  
  For individuals with business addresses, the flow identifies the correct Organization Name using relationships in this priority:  
    1. Employer  
    2. Principal/Director  
    3. Employee (catch-all)

- **Distinct Letter Headers – 4 constituent scenarios**

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
      *(1. Primary Contact → 2. Principal/Director → 3. Employee as catch-all)*  
    - Business Name  
    - Business Address  

{{< /fold >}}


1. {{< fold title="Getting Started: SKY API and Power Automate setup" >}}
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

   We create a for loop using Apply to each by passing the full array output from List Gifts into Apply to each:
   `@outputs('List_gifts')?['body/value']`

   Most of the work after this happens inside the loop, where we make additional calls to gather information about each gift.

   {{< /fold >}}

4. {{< fold title="Inside Apply to each and retrieving gift information" >}}

   Our first call inside each iteration is Get a gift. We make this call to get gift date, gift amount, constituent ID, and appeal ID.

   This call requires a gift ID, which comes from List Gifts:
   `items('Apply_to_each')?['id']`

   {{< /fold >}}

5. {{< fold title="Retrieving constituent information" >}}

   We call Get a constituent using the constituent ID from Get a gift.
   This provides title, first name, last name, and address.

   {{< /fold >}}
