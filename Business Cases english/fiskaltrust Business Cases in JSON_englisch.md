Introduction
============

Fiskaltrust Middleware translates real life business cases into the German
Financial Adminstration’s definitions and requirements of its digital interface
for cash desk systems (Digitale Schnittstelle der Finanzverwaltung für
Kassensysteme, DSFinV-K).

Cash desk producers stay focussed on their needs – fiskaltrust ensures proper
fiscalisation.

All you need is configuring a “cash box” in our fiskaltrust portal (website) and
download the Middleware to your cash desk. Fiscalisation of transactions will be
achieved by requests to our Middleware.

The “cash box” works as a container for your configuration; you choose and
specify technical security equipment (TSE) to be used for signing and
fiscalising receipts next to adding features (“helpers”), e.g. send requests to
your cash box in “REST”-style (instead of using SOAP).

This document gives examples for typical business situations in common branches
and translates these into JSON code calls of our middleware.

### Document status

| Rev | Modification                               | modified by | Date of completion |
|-----|--------------------------------------------|-------------|--------------------|
| 00  | Creation, principles, first JSON cases     | Lars Mach   | Jan 20, 2020       |
| 01  | JSON business cases for various industries | Team IT     |                    |
| 02  | Review                                     | pending     |                    |

Principles
==========

### Script language: JSON or XML

Calls to our service can be sent either in XML or JSON format.

JSON (Java Script Objection Notation) is characterised by a slim syntax, while
XML provides more flexibility such as using hexadecimal numbers, which JSON does
not.

### Using our middleware as service

Calling our middleware involves the use of certain (hexadecimal-based) variables
to trigger functions described in our interface documentation, some of which can
contain flags. These flags must be set within those variables, e.g. using
Boolean OR-operations on variables and flags (both in binary format) or simply
adding the flag-value to the un-flagged variable (in any numeral system).

### Own additional fields (branch or case specific)

Our middleware offers containers (string type fields) to create additional
fields – these can be named by own preference, e.g. customer name/VAT ID for
invoices in “cbCustomer” or hotel reservation dates etc. in
“ftChargeItemCaseData”. Field names and values must be written into such
container fields using subordinated JSON script (see next paragraph).

### How to use optional fields for subordinated JSON scripts

Some of the interface fields are used as containers for subordinated JSON code;
for example: cbUser, cbCustomer, cbArea, ftChargeItemCaseData and others (see
interface documentation for more).

You can define own field names and values therein, embedded in subordinated
JSON, e.g.:

>   “cbCustomer”: “{\\”company\\”: \\”fiskaltrust\\”, \\”VAT-No\\”:
>   \\”DE323821961\\”, \\”YourNumber\\”: 123 }”

>   NB: Backslashes are needed to insert subordinated quotes properly.

This writes the following script into cbCustomer (formatted):

{

“Company”: “fiskaltrust”,

“VAT-No”: “DE323821961”,

“YourNumber”: 123  
}

### Embedding JSON script in XML

Some container fields require JSON format. If you use XML to send requests to
the cash box, then above JSON code example in the “cbCustomer” field will look
like this:

\<cbCustomer\>

\<![CDATA[ { ”Company”: ”fiskaltrust”, ”VAT-No”: ”DE323821961”, ”YourNumber”:123
} ]]\>

\</cbCustomer\>

### Implicit vs. explicit flow

German rules call for a fiscalisation not only of cash register receipts, but
including all transactions belonging to a process (e.g. booking a room, issuing
delivery notes, adding drinks to a room bill).

An explicit workflow described by the DSFinV-K requires to initiate a process by
a start transaction, followed by one or more updates, and finished by requesting
a receipt.

There are restrictions in connection with open processes (e.g. maximum number of
open processes in a TSE at a time and the need to close processes at daily cash
register closing). This imposes challenges, particularly on very long process
types like hotel bookings months in advance with services added after check-in,
before issuing a final receipt.

The fiskaltrust middleware offers an easy approach to such:

When calling our middleware to generate different receipt types (ftReceiptCase),
you can set a flag (0x100000000) in order to circumvent above issues: We call
this “implicit flow”.

The implicit flow needs just ONE call, which makes our middleware send two calls
to start and close a transaction at once – no open process will be left in the
TSE.

Define a process by reference numbers:

Simply define “cbReceiptReference” for the first transaction of a process and
make all following ones belong to the same by pointing to that
cbReceiptReference, using cbPreviousReceiptReference.

If adding date and time information to cbReceiptReference when starting a
process in implicit flow (e.g. “ClientXYZ\#20200211-112430” for a process that
started on Feb 11th, 2020 at 11:24:30h), then you can refer to the start of that
process by setting cbPreviousReceiptReference to that first reference when
issuing a receipt. This format is not mandatory, but a recommendation.

In this way you can easily specify a process that lasts several months without
experiencing technical challenges or restrictions by the TSE, and all
transactions belonging to this process will be marked.

You will find JSON call examples using the implicit flow in this document.

### Cancellation or correction of receipts

The idea of fiscalisation is manipulation-protected storage of receipts; any
changes or nullification thereafter will be impossible.

This needs a different approach to cancellations or changes of entire receipts
or parts thereof.

If fiscalised receipts or positions therein must be revoked or adjusted, then
new “counter”-receipts have to be generated with relevant items and payments set
to negative values. Those “counter”-receipts can use cbPreviousReceiptReference
to refer to the original receipt or transaction.

Such concept also applies when splitting an order (guests sharing a restaurant
table ask for separate receipts) or when merging transactions. Items to be
transferred from the original process (“all guests at the table”) are set in
negative values, and new processes are generated (“one per guest”) to which
these items are moved.

You will find handy examples of such situations in our JSON code collection in
this document.

### Header data used in all JSON examples

*All JSON code examples in this document use these identical demonstration
values (IDs):*

ftCashBoxID: c094f242-91d5-4343-9c54-bce85f70d0d6 (fiskaltrust.cashbox ID
assigned by user portal)

cbTerminalID: CashDeskMaker_Model_1 (unique ID [UID] specifying POS/terminals)

ftPosSystemId: b3dc6573-96d9-e611-80f7-5065f38adae1 (optional UID specifying POS
software version)

JSON Calls
==========

Function calls
--------------

### Register new cash desk 

Each cash box (POS) must be registered once in life before use, using the below
call:

| fiskaltrust.ifPOS.v0 { ReceiptRequest { "ftReceiptCase": 4919338172267102211, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMaskerModel_Serial123“, "cbReceiptReference": "PutYourOwnReferenceHERE", "cbReceiptMoment": "2019-10-25T13:32:45.133Z", "cbChargeItems": [], "cbPayItems": [] } } | Comments i.e. 0x4445 0001 0000 0003 “initial operation receipt” (implicit flag mandatory) |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|


### Remove (decommission) cash desk

A cash desk (POS) can be decommissioned forever; this action cannot be reversed.

| fiskaltrust.ifPOS.v0 { ReceiptRequest { "ftReceiptCase": 4919338172267102212, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMaskerModel_Serial123“, "cbReceiptReference": "PutYourOwnReferenceHERE", "cbReceiptMoment": "2019-10-25T13:32:45.133Z", "cbChargeItems": [], "cbPayItems": [] } } | Comments i.e. 0x4445 0001 0000 0004 “out of operation receipt” (implicit flag mandatory) |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------|


Shops: Cash register situations
-------------------------------

### Scanner check out

Scanner check outs collect charge items in short time and collect payments
(cash, card) immediately.

According to the DSFinV-K (Appendix H) rules of simplification apply: The
process will start with the charge first item scanned, and no updates will be
required for each following one. The process is closed by a POS receipt (with
all remaining charge items listed therein).

This applies to all similar cash desks (e.g. manual collection of charge items
through a keyboard).

Example:

Explicit flow; request 1) process started with first item (1x dress) and 2)
closed (with all remaining items: 1x jeans, 2x shirts) and specified payment
type (here: cash).

| { "ftReceiptCase": 4919338167972134920, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMasker_Model_1", "cbReceiptReference": "SALE_20200204\#11354513", "cbReceiptMoment": "2020-02-04T11:35:45.133Z", "cbArea": "Outlet_47798_Krefeld", "cbChargeItems": [ { "ftChargeItemCase": 4919338167972134929, "Description": "Dress_Aztecs-style_black-AOP", "Quantity": 1.0, "Amount": 119.99, "VATRate": 19.00, "VATAmount": 19.16, "ProductGroup": "Clothing_women", "ProductNumber": "18.908.82.9816.99A0.42", "ProductBarcode": "4062033264091", "Unit": "Article", "UnitQuantity": 1, "UnitPrice": 119.99, "Moment":"2020-02-04T11:35:45.133Z" } ], "cbPayItems": [] }                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | *First request:* HEX 0x445000000000008 i.e. „start transaction” cbReceiptReference specifies a process with ID “SALE_20200204\#11350013” (“11350013” as time stamp) HEX 0x4445000000000011 i.e. “delivery normal, 19%” First charge item is listed:                                                                                                                                                                              |
|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | A black dress; price €119.99 Your internal product code Barcode, e.g. European Article Number (EAN) Time stamp of scanning No payments yet; however, empty array is mandatory                                                                                                                                                                                                                                                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| { "ftReceiptCase": 4919338167972134913, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMasker_Model_1", "cbReceiptReference": "SALE_20200204\#11360013", “cbPreviousReceiptReference”: “SALE_20200204\#11354513”, "cbReceiptMoment": "2020-02-04T11:36:30.133Z", "cbArea": "Outlet_47798_Krefeld", "cbChargeItems": [ { "ftChargeItemCase": 4919338167972134929, "Description": "Jeans_Flared_leg", "Quantity": 1.0, "Amount": 79.99, "VATRate": 19.00, "VATAmount": 12.77, "CostCenter": "1", "ProductGroup": "Clothing_women", "ProductNumber": "18.003.71.9238.57Z4.40.32", "ProductBarcode": "4061956300718", "Unit": "Article", "UnitQuantity": 1, "UnitPrice": 79.99, "Moment": "2020-02-04T11:36:10.133Z" }, { "ftChargeItemCase": 4919338167972134929, "Description": "Jerseyshirt_V-neck_terracotta", "Quantity": 2.0, "Amount": 35.98, "VATRate": 19.00, "VATAmount": 5.7447, "ProductGroup": "Clothing_women", "ProductNumber": "18.905.32.9415.2745.40", "ProductBarcode": "4062033036155", "Unit": "Article", "UnitQuantity": 1, "UnitPrice": 17.99, "Moment": "2020-02-04T11:36:10.133Z" } ], "cbPayItems": [ { "ftPayItemCase": 4919338167972134913, "Amount": 235.96, "Description": "Bar", "Moment": "2020-02-05T11:36:45.133Z" } ] } | *Second request:* HEX 0x4445000000000001 i.e. “pos-receipt” Reference of current call                                                                                                                                                                                                                                                                                                                                            |
|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Reference to start of process Remaining charge items… HEX 0x4445000000000011 i.e. delivery normal 2 identical items scanned Gross price of all 2 items Total VAT amount of 2 items Price per shirt HEX 0x4445000000000001 i.e. cash payment EUR                                                                                                                                                                                  |

### Scanner check out – with corrected erroneous scanning

If wrong items are scanned during the above process “scanner check out”, then
these can be removed at once in the cbChargeItems list by counter-items with
negative values.

Example:

Explicit flow; request 1) process started with first item (1x yoghurt) and 2)
the bar code of a bottle in a six pack is scanned – and removed again, because
the six pack has got an own bar code (and lower total price); the correct amount
is paid by debit card.

| { "ftReceiptCase": 4919338167972134920, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMasker_Model_1", "cbReceiptReference": "SALE_20200204\#11354513", "cbReceiptMoment": "2020-02-04T11:35:45.133Z", "cbArea": "Supermarket_47807_Krefeld", "cbChargeItems": [ { "ftChargeItemCase": 4919338167972134930, "Description": "Yoghurt_greek_almond", "Quantity": 1.0, "Amount": 0.69, "VATRate": 7.00, "VATAmount": 0.04514, "ProductGroup": "FOOD_milk", "ProductBarcode": "4025500170035", "Unit": "Article", "UnitQuantity": 1, "UnitPrice": 0.69, "Moment":"2020-02-04T11:35:45.133Z" } ], "cbPayItems": [] }                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | *First request:* HEX 0x445000000000008 i.e. „start transaction” cbReceiptReference specifies a process with ID “SALE_20200204\#11350013” HEX 0x4445000000000012 i.e. “delivery discounted 7%” First charge item is listed: Greek yoghurt; price €0.69 Your internal product code Barcode, e.g. European Article Number (EAN) Time stamp of scanning No payments yet; however, empty array is mandatory                                                                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| { "ftReceiptCase": 4919338167972134913, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMasker_Model_1", "cbReceiptReference": "SALE_20200204\#11360013", “cbPreviousReceiptReference”: “SALE_20200204\#11354513”, "cbReceiptMoment": "2020-02-04T11:36:30.133Z", "cbArea": "Supermarket_478097_Krefeld", "cbChargeItems": [ { "ftChargeItemCase": 4919338167972134930, "Description": "XYBrand_Lemonade", "Quantity": 1.0, "Amount": 1.29, "VATRate": 7.00, "VATAmount": 0.08439, "ProductGroup": "FOOD_drinks_nonAlc", "ProductBarcode": "4061956300718", "Unit": "Bottle", "UnitQuantity": 1, "UnitPrice": 1.29, "Moment": "2020-02-04T11:36:10.133Z" }, { "ftChargeItemCase": 4919338167972134930, "Description": "XYBrand_Lemonade", "Quantity": -1.0, "Amount": -1.29, "VATRate": 7.00, "VATAmount": -0.08439, "ProductGroup": "FOOD_drinks_nonAlc", "ProductBarcode": "4061956300718", "Unit": "Bottle", "UnitQuantity": 1, "UnitPrice": 1.29, "Moment": "2020-02-04T11:36:10.133Z" }, { "ftChargeItemCase": 4919338167972134930, "Description": "XYBrand_Lemonade_6pack", "Quantity": 1.0, "Amount": 6.50, "VATRate": 7.00, "VATAmount": 0.08439, "ProductGroup": "FOOD_drinks_nonAlc", "ProductBarcode": "4061956300728", "Unit": "SIXPACK", "UnitQuantity": 1, "UnitPrice": 6.50, "Moment": "2020-02-04T11:36:10.133Z" } ], "cbPayItems": [ { "ftPayItemCase": 4919338167972134916, "Amount": 7.19, "Description": "Sparkasse_PIN", "Moment": "2020-02-05T11:36:32.133Z" } ] } | *Second request:* HEX 0x4445000000000001 i.e. “pos-receipt” Reference of current call                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Reference to start of process Remaining charge items… HEX 0x4445000000000012 i.e. “delivery discounted 7%” 1x bottle of lemonade… …was scanned wrongly! HEX 0x4445000000000012 i.e. “delivery discounted 7%” Negative numbers: 1x bottle of lemonade… …is removed from list! (e.g. EAN of bottle scanned instead of six-pack’s EAN) UnitPrice remains positive HEX 0x4445000000000012 i.e. “delivery discounted 7%” Six-pack of lemonade added HEX 0x4445000000000004 i.e. debit card payment                                                                                      |

Delivery notes
--------------

### Create a delivery note

According to German “Kassensicherungsverordnung” (KassSichV) business cases such
as delivery notes must be fiscalised.

This is accomplished by a receipt request “delivery note” that contains charge
items but no pay item.

Example:

Implicit flow – three items (1x funghi, 2x margherita pizzas) are acquired
(keyboard, scanner), then a delivery note is fiscalised by sending the following
request to our middleware:

| { "ftReceiptCase": 4919338172267102223, "ftCashBoxID": "c094f242-91d5-4343-9c54-bce85f70d0d6", "ftPosSystemId": "b3dc6573-96d9-e611-80f7-5065f38adae1", "cbTerminalID": "CashDeskMasker_Model_1", "cbReceiptReference": "DELIVERY_234_20200204\#113410013", "cbPreviousReceiptReference": "ORDER_234_20200204\#112022008", "cbReceiptMoment": "2020-02-04T11:43:00.133Z", "cbArea": "PizzaDeliveryShop_Cologne_02", "cbChargeItems": [ { "ftChargeItemCase": 4919338167972134930, "Description": " Pizza_Funghi", "Quantity": 1.0, "Amount": 0.0, "VATRate": 7.00, "VATAmount": 0.0, "ProductGroup": "Speisen_ausserHaus", "ProductNumber": "06", "UnitPrice": 6.00, "Moment":"2020-02-04T11:42:45.133Z" }, { "ftChargeItemCase": 4919338167972134930, "Description": " Pizza_Margherita", "Quantity": 2.0, "Amount": 0.0, "VATRate": 7.00, "VATAmount": 0.0, "ProductGroup": "Speisen_ausserHaus", "ProductNumber": "01", "UnitPrice": 4.00, "Moment":"2020-02-04T11:42:52.133Z" } ], "cbPayItems": [] } | Comment: HEX 0x444500010000000F i.e. delivery note (implicit) Reference ID delivery note Reference to pizza order HEX 0x4445000000000012 i.e. delivery discounted 7% HEX 0x4445000000000012 i.e. delivery discounted 7% |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

