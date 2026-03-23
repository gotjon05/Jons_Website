+++
date = '2026-01-30T12:42:04-05:00'
draft = false
title = 'Automating Non-profit Acknowledgement Letters With Blackbaud API (Work in Progress)'
+++

This is a walkthrough for automating the creation of acknowledgement letters for Non-profits with Blackbaud NXT, using Blackbaud SKY API, Power Automate and Sharepoint. 

As we build the flow, I will break down each Power Automate action, look at the JSON returned by our API calls, and show how WDL expressions are used to access and extract the specific data we need at each step.

**What this Accomplishes:**

This workflow ensures that on a set schedule, every donor receives the right acknowledgment—addressed to the right person, at the right address, using the right letter template—without manual intervention.

{{< fold closed="The business rules that guided this process flow" >}}

- **Gifts considered in this Flow**  
  Only gifts greater than zero, not yet acknowledged, and not unpaid pledges.

- **Letter Template Matching Rule**
  A gift’s **Appeal + Package** combination determines which letter template/content is used.

- **Business Identity Rule**  
  For individuals with business addresses, the flow includes the Company Name by identify the correct Business Name using the constituent’s Primary Employer relationship.

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

{{< fold closed="Overview" >}}

In a single loop through every unacknowledged gift:

1. **Gather fields required for the Letter**

- addressee of Hard Credit/Soft Credit title, First Name, Last Name, Address (street1, city, state, ZIP) using *Get Constituent*
- Gift Amount, Gift Date using *Get Gift*
- Business Name using *List constituent relationships*
  
2. **Set Flags for header and addressee logic**
   
  We use Compose Actions as flags to determine how the letter header and addressee should be built later in the process. 

  - Whether the gift includes a soft credit
  - Whether the soft-credit recipient has a preferered home or business address
  - Whether the constituent is an organization or an individual
  - Whether the hard credit recipient has a preferered home or business address
  - Whether the recipient is a foundation
  

3. **Map Appeal + Package combinations to Lettercodes**
   
  I use information from a gifts Appeal + Package to decide what template to use. But because of overlapping Appeal + Package combinations for one acknowledgement template, I created a LetterCode dictionary that groups overlapping Appeal + Package combinations and maps them to the appropriate letter template.
   
4. **Generate dynamic first paragraphs where needed**
   
  Some letters share identical content except for the first paragraph, which varies by event. Rather than creating separate templates for each variation, the first paragraph is generated dynamically while the rest of the letter remains fixed. A dictionary maps Appeal + Package combinations to their corresponding paragraph, allowing the correct paragraph to be retrieved.

5. **We use a nested if statement for each element of the header with the Order of priority: Soft Credit → Foundation → Individual**

6. **Construct Header into one compose, pulling from each element of the header in previous step**

7. **Create Word templates stored in SharePoint**
  
  Word templates are stored in SharePoint and referenced by the Switch action, which selects and populates the appropriate template using the LetterCode.

8.  **Switch Action with LetterCode as input to match Header/Salutations information with the correct template**

9.  **Mark the Gift as acknowledged**

10.   **Email the Donor**

11.  **Create Labels for mailing each Letter**
{{< /fold >}}


1. {{< fold title="Getting Started: SKY API and Power Automate setup" >}}

  1. [To get Power Automate Premium $15/Month as of 2/14/26](https://go.microsoft.com/fwlink/?linkid=2252273&clcid=0x409&culture=en-us&country=us)

  2. [Go to Blackbaud Marketplace and Connect with Power Automate](https://app.blackbaud.com/marketplace/applications/848afa1d-fe4a-4331-8e03-79ea0d000343?appdetails-active-tab=overview)

  3. Create a scheduled cloud flow in Power Automate

  4. Click the + sign and Search for Blackbaud Raisers Edge NXT List Gifts

  5. Blackbaud connector handles authentication internally and you will be prompted to sign-in with your Blackbaud account 
  {{< /fold >}}

2. {{< fold title="Retrieving list of all Unacknowledged gifts">}}
  Our first SKY API call is [**List Gifts**](https://developer.sky.blackbaud.com/api#api=58bdd5edd7dcde06046081d6&operation=ListGifts), which retrieves all unacknowledged gifts from Raiser’s Edge NXT in a Json Array. 

  1. Click on List Gift and update the parameters:

  2. Start gift amount: 1 --- We want gifts that are greater than

  3. Acknowledgement status: NotAcknowledged

  4. Added on or after: (Optional, if your database is messy, you can select the earliest date for gifts you want. But you need a specific date format.) SELECT Fx and enter:   
```
  formatDateTime('2025-05-10', 'yyyy-MM-ddT00:00:00Z')
```
  5. Type: Enter the the gift types below to avoid Pledges: 
 ```
 Donation,GiftInKind,MatchingGiftPayment,PledgePayment,RecurringGiftPayment,Stock,SoldStock
 ```  
   {{< /fold >}}


3. {{< fold title="Looping through List Gifts" >}}

We loop through the output of List Gifts because each subsequent Blackbaud API call requires one unique key. 

**Get a Gift** requires the unique Gift ID of each Gift. **Get Constituent** requires the Constituent ID of each constituent hard credited for that gift.

So the full workflow of retrieving gift + donor details, selecting the right template, and generating the letter runs once per gift, inside each iteration of the loop.

  1. Add the action "apply to each", and and set the input to `body('List_Gifts')?['value']`
  Alternatively, you can select Dynamic content (lightning icon) to select body/value from List Gifts. 
  
{{< fold title="Why we use body('List_Gifts')?['value'] as input in apply to each">}}
After running your Power Automate Flow and opening the Raw Output of List Gifts, We will See:
- The output as one big JSON object, defined by the enclosed outside bracket "{}".  
- Inside this object are two other objects. **headers** and **body**. 
You will notice that the **body** object has the relevant data we need inside the JSON array called **value** defined by the enclosed "[]"

Thats why we loop through `body('List_Gifts')?['value']` 

This is our first use of a Power Automate expression to access and extract data. 
  - We use the body() function because it selects the body object we want
  - To safely access the JSON array **value** to extract it, we use `?` in `?['value']`. This insures that body('List_Gifts') returns NULL if value doesnt exist or is NULL so it doesnt break. 

  (I initially used `outputs('List_Gifts')?['body/value']` but later changed it to `body('List_Gifts')?['value']`. They point to equivelent paths but body is more direct, reducing the chance of referencing the wrong thing. Output returns the entire response while body returns the value portion.) 

{{< /fold >}}

{{< fold title="Why we Loop instead of Parse through List Gifts">}}

While we could parse the List Gifts output to use any fields it already includes, it provides key IDs (Gift ID, Constituent ID) but not all letter-ready details we need. For that reason, we still would need a For each loop to make these individual calls per gift.
{{< /fold >}}

{{< /fold >}}




4. {{< fold title="Retrieving gift information for each gift in List Gifts" >}}

  Our first call inside our **For Each** loop is **Get a gift**. This call returns the fields we
  need for each record: gift date, gift amount, constituent ID, and appeal ID.

  **Get a gift** requires a gift ID as an argument. As **For Each** is looping over the value array returned by **List Gifts**, we can pull the gift ID from each iteration of gifts inside the loop and pass it as an argument to **Get a gift**.


  1. Expand "Apply to each" and select the + icon

  2. Search for blackbaud NXT get a gift

  3. Add the action, then enter the expression `items('Apply_to_each')?['id']` **or** select Dynamic content (lightning icon) and look for system record id of gift from List gifts


{{< fold title="Why we use items() instead of body() or output() as an argument in Get a Gift" >}}
  We use the function **items()** because it returns the current item in a loop. 
  
  We dont use **body()** or **output()** because control containers like **For Each** dont have an Output or a Body. We are going to introduce more Control Containers like Conditions and Switch later in the Flow. 
{{< /fold >}}
  In the next step, we are going to get Constituent information using the output of Get a Gift

   {{< /fold >}}

 
5. {{< fold title="Retrieving constituent information for each gift in List Gifts" >}}
 
List_Gifts now calls **Get a Gift** for each id in the body of List Gifts. Inside the output we can find the Constituent ID key of the Hard Credit Constituent who was credited for the gift. Using this Constituent_ID with **Get Constituent** will provide Title, First Name, Last  Name and address information.

  1. Inside the For Each Loop, lets add blackbaud NXT call **Get a Constituent** after Get a gift.  

  2. Constituent ID is a required field and we need to retrieve it from the output of our previous call **Get a Gift** using body('Get_a_gift')?['constituent_id'] 

Now in every iteration of For Each(), we are retrieving gift and constituent information for each gift. 

{{< /fold >}}

6. {{< fold title="Checking the Type of Constituent">}}
  We use several Compose actions as flags to determine what information must be retrieved and how the letter header should be constructed. 
  
These flags identify:

  - Whether the hard credit is an organization
  - Whether the gift includes a soft credit
  - Whether the hard or soft credited constituent uses a home or business address
  - Whether the hard credit is a foundation

1. Create a Compose to check if Gift has a Soft Credit and call it CheckIfSoftCredit. This Compose returns true when the gift includes at least one soft credit and false otherwise

Pseudocode:
  ```text
  If Soft Credit is not empty
    return true
  Else 
    return false
  ```
{{< fold title="Code for Determining if Gift has a Soft Credit" >}}
```not(empty(body('Get_a_gift')?['soft_credits']))```
{{< /fold >}}

2. Create compose to check if soft credit of gift has a prefered address of home or buisness. Call it IndividualHomeOrBusinessAddress

Pseudocode:
  ```text
  If Constituent CheckIfSoftCredit is true
    If address is type Home and preferred is true
      return 'home'
    Else If Preferred address is type business and preferred is true
      return 'business'
    Else return ''
  Else 
    return ''
  ```
{{< fold title="Code for Determining if a soft credits preferred address is home or business" >}}
```
if(
  outputs('CheckIfSoftCredit'),
  if(
    and(
      equals(toLower(coalesce(outputs('Get_a_constituent')?['body/address']?['type'], '')), 'home'),
      equals(coalesce(outputs('Get_a_constituent')?['body/address']?['preferred'], false), true)
    ),
    'Home',
    if(
      and(
        equals(toLower(coalesce(outputs('Get_a_constituent')?['body/address']?['type'], '')), 'business'),
        equals(coalesce(outputs('Get_a_constituent')?['body/address']?['preferred'], false), true)
      ),
      'Business',
      ''
    )
  ),
  ''
)
```
{{< /fold >}}

3. Create compose to check if hard credit of gift has a prefered address of home or business. Call it IndividualHomeOrBusinessAddress

Pseudocode:
  ```text
  If Constituent is individual
    If address is type Home and preferred is true
      return 'home'
    Else If Preferred address is type business and preferred is true
      return 'business'
    Else return ''
  Else 
    return ''
  ```
   
{{< fold title="Code for Determining if a hard credits preferred address is home or business" >}}

```
if(
  outputs('CheckIfSoftCredit'),
  if(
    and(
      equals(toLower(coalesce(outputs('Get_a_constituent')?['body/address']?['type'], '')), 'home'),
      equals(coalesce(outputs('Get_a_constituent')?['body/address']?['preferred'], false), true)
    ),
    'Home',
    if(
      and(
        equals(toLower(coalesce(outputs('Get_a_constituent')?['body/address']?['type'], '')), 'business'),
        equals(coalesce(outputs('Get_a_constituent')?['body/address']?['preferred'], false), true)
      ),
      'Business',
      ''
    )      
  ), 
  ''
)   
```    
{{< /fold >}}

4. Cje


 



      
{{< /fold >}}

1. {{< fold title="Retrieving Soft Credit Information">}}
  


  If the Flag checks for Soft Credit returns True, 

We first check whether the gift includes a soft credit. If a soft credit exists, the acknowledgment letter should be addressed to the soft credit individual rather than the hard credit donor. We retrieve the title, first name, and last name of the soft credit individual so the correct addressee can be constructed in the letter header.

   1. **Check if gift includes a soft credit**
       We add a condition action that checks if Soft Credit is Present in 
   2. 
{{< /fold >}}



8. {{< fold title="Get Appeal and Package" >}}
  We retrieve the Appeal + Package for each gift because this combination determines which letter template we use.To do this safely, we add a Condition that checks whether an Appeal ID exists in each gift. If it does, we pull the appeal_id from Get a Gift


{{< fold title="What is a Condition? And Why is it relevant?" >}}  
  A **Condition** in Power Automate is similar to an if/then statement. It evaluates an expression as True or False, often using AND or OR. 

  In this flow i have use **Conditions** to prevent running actions that depend on values that may be **null** (missing). Without these safeguards, Power Automate could attempt to call a connector with a blank ID (or reference missing data), which can cause the action—and sometimes the entire run—to fail.
{{< /fold >}}

{{< /fold >}}
9. {{< fold title="Resolving overlapping Appeal+Package Letter Codes with Template Part 1" >}}
The Appeal + Package Code from a gift determines which letter template to use.

But in this use case, the relationship was many-to-one. Many different Appeal + Package combinations needed to map to the same letter template. For example, the “General” letter template had 5+ different Appeal + Package combinations associated with it.

And this was a problem because later in the flow, I use a single code to determine which template is generated for each gift, so Appeal + Package alone wasn’t a reliable “single selector. 

In order to have a single code to indicate what template to use for each gift, I introduced LetterCodes that are 1-to-1 with templates. With each LetterCode having an array of Appeal+Package codes that share the same LetterCode and template. 

To store this mapping, I created a JSON lookup table (an array of objects) where each object contains a LetterCode and an array of ass ociated Appeal + Package combinations.

And for each gift, i scan the appeal+Package code and returned the matching LetterCode. 

{{< fold title="Understanding the JSON table below" >}}
This may not look like an Excel Table with the JSON syntax but the pattern represents the same columns and rows of Excel. 

Each {} represent each object or row. In this example, i have 2 objects/rows. 

And each Object has two name/value pairs, "LetterCode" is a name with "Gala" as a value. "AppealPackages" is a name with an array of appeal+Package codes as a value.

And because we have more than one object, its a list of Objects, whichs needs an outside bracket []

{{< /fold >}}

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


10. {{< fold title="Resolving overlapping Appeal+Package Letter Codes with Template Part 2" >}}
{{< /fold >}}



11. {{< fold title="Identifying the type of Constituent to provide the correct Header and salutation later" >}}
{{< /fold >}}

12. {{< fold title="Creating Letters, as Adobe PDF's or Word Documents, and Marking Gifts as Acknowledged" >}}
{{< /fold >}}

13. {{< fold title="Emailing Acknowledgement Letters to Donors" >}}
{{< /fold >}}


{{< fold title="Review and Maintaining this Process" >}}
{{< /fold >}}
