+++
date = '2026-09-30T00:00:00-07:00'
draft = false
title = 'Automating Donor Acknowledgment Letters Across 21 Hospitals'
+++


## Goal:
Automate the creation of donor acknowledgment letters for 21 hospitals by combining Raiser’s Edge NXT gift and constituent data with hospital-specific configuration, approximately 280 letter variations, dynamic gift content, and recipient-specific addressing rules.

## Business Rules:
- Gifts Letter Code determines the content of the letter; the gift's Constituent Code determines hospital-specific information.
- Header requires hospital-specific information: hospital/foundation name, address, phone, email, and website.
- The content of the letter needs to use one of approximately 280 existing letters specifically written by the hospitals.
- Letter content includes dynamic values: `{giftamount}`, `{giftdate}`, `{giftfund}`, `{gifttribute}`, `{giftpledgeamount}`, `{recurringgiftfrequency}`, `{recurringgiftamount}`, `{giftreceiptamount}`, `{giftinkind}`, and `{addressee}`.
- Addressee rules: if the hard credit is an organization, use Receipts Contact + Organization Name; otherwise, use Organization Name. For individuals, use the addressee of the hard-credit individual. If the Letter Code indicates a soft-credit scenario and the gift contains a soft credit, use the addressee from the soft-credit constituent.
- Salutations need to reflect the selected recipient.
- Signature and footer are specific to the market of the gift.
- Letter filename: Batch Number + Letter Code + Date + Constituent Name + Timestamp + `.docx`.
- Tax/receipt language is included in the footer, depends on the Letter Code, and can contain dynamic content.
- Three hospitals were excluded from acknowledgment-letter generation but could still have recurring gifts entered into the database.





## Design:
The data for the letters came from two different sources. Hospital-specific information and letter content were static information that could not be retrieved from the database, so I gathered and stored them in JSON configuration objects. Dynamic gift and constituent information, such as gift amount, fund, date, addressee, salutation, tribute, recurring gift information, and soft-credit data, was retrieved from Raiser’s Edge NXT through SKY API calls. The Letter Code and Constituent Code from the gift, were crucial for connecting the appropriate hospital and letter specific information. 

During this project, the template and logo of the letter was standardized for all 21 hospitals. There was agreement in principle to standardize lettercodes to reduce the amount but wasn't implemented. The existing lettercodes were doing too much, capturing market specific information and Gift specific information. If lettercode were to only be gift specific, the number could be brought down to 117 distinct lettercodes. The market specific information would be directly sourced from the gifts constituent code. 

My solution to handling the large amount of LetterCode specific content and the Hospital specific information was to store them in two Nested JSON objects. I used the Letter Code and Constituent Code as keys because they reliably identified the correct acknowledgment content and hospital configuration for each gift. Using bracket notation, the flow could dynamically look up the matching key and return only the values needed for that gift. These values stored the static information that could not be retrieved directly from the database, such as letter content, tax language, and Hospital specific information.

Inside the static content of my nested json objects were placeholders for all the dynamic language i needed. After retrieving the content from my nested json objects, I replaced the placeholders with their actual values. 

I also gathered the constituent and gift information needed to determine how each letter should be addressed. I created flags to identify the relevant recipient scenario, including an individual hard-credit constituent, an organization hard-credit constituent with a designated Receipts Contact, or a DAF/soft-credit scenario. Those flags allowed the flow to apply the appropriate addressee and salutation rules when building the letter.

Nested JSON of Hospital/Constituent Code Dictionary:
MarketName: hospital/foundation name
MarketInformation: hospital-specific contact information such as address, phone, email, and website
Footer1: market-specific closing/valediction
Footer2: additional signer/footer information
Signature: the signature file/configuration used for that market

Nested JSON of LetterCode:
content: the body text of the acknowledgment letter
markettaxID: the tax/receipt language associated with that letter code



## Workflow

1. **Identify gifts that require acknowledgement**
   1. Use **List Gifts** to retrieve an array of unacknowledged gifts.
   2. Loop through each gift.

2. **Retrieve Gift Specific Hospital information**
   1. Retrieve the gift and constituent record.
   2. Use the gift's Constituent Code to identify the corresponding hospital configuration.
   3. Retrieve the hospital-specific values from the Hospital/Constituent Code Dictionary.
   4. Skip gifts associated with excluded hospitals.

3. **Determine Addressee and Saluation**
   1. Retrieve name-format information for the addressee and salutation.
   2. Check organization relationships for a Receipts Contact.
   3. Retrieve soft-credit information when applicable.
   4. Retrieve tribute, fund, receipt, recurring-gift, pledge, and Gift-in-Kind information as needed.

4. **Retrieve Content corresponding to the gifts Letter Code**
   1. Use the gift's Letter Code to retrieve the corresponding letter content.
   2. Retrieve the applicable tax and receipt language.

5. **Apply business rules and construct dynamic content**
   1. Create flags identifying the applicable gift and recipient scenarios.
   2. Apply the addressing rules to determine the addressee, salutation, and organization values.
   3. Store the dynamic replacement values in the Tokens Compose.
   4. Replace the placeholders in the letter content with the corresponding token values.

6. **Generate and complete the acknowledgement**
   1. Populate the Word template.
   2. Save the generated letter to SharePoint.
   3. Log the output.
   4. Mark the gift as acknowledged.

















