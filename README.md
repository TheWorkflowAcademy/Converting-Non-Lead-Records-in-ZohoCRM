# Converting-Non-Lead-Records-in-ZohoCRM
A Deluge script that replicates the conversion of a record across different modules in Zoho CRM by copying info to a new record in a different module and deleting the old record.

## Core Idea
Zoho CRM allows users to convert records from Leads to Contacts on a click of a button, however it is only limited to the said module. This script aims to replicate the conversion button should you need to convert records across modules that aren't Leads to Contacts. This is done by copying all the necessary fields and related lists from the original module to the new, then deleting the old record.

## Example Case
Your insurance business has two different types of client accounts, "Households" and "Businesses". Due to unique requirements, they are managed in CRM with two modules, one for each type. At any point of time, your staff needs to convert "Household" records to "Businesses" and vice versa. In this demonstration, we will show you how to convert a records from Households to Businesses. For reverse conversion, the script will have to be modified accordingly.
* Another popular use case is to "Undo Convert" of Contacts (i.e. convert a Contact back to a Lead). [Click here to jump straight to that.](#example-use-case---convert-contacts-back-to-leads)

## Configuration
* The two inter-related conversion modules should be set up with the relevant fields mapped.
* The script should be written in a custom button for each module so users can easily execute the conversion from the CRM record page.
* Zoho CRM Connection needed:
  * ZohoCRM.modules.ALL

## Tutorial
### Get the Old Record Fields and Create a New Record
Get all necessary fields from the original (old) record, then create a new record in the destination module. Get the record ID from the `createRecord` function for updates later on in the script.

```javascript
//Change below to the relevant old & new modules
oldModule = "Households";
newModule = "Businesses";

oldRecord = zoho.crm.getRecordById(oldModule,recordId);

//Get all Necessary Fields - add/modify as needed
recordName = oldRecord.get("Name");
email = oldRecord.get("Primary_Email");

//Create Record
rMap = Map();
rMap.put("Name",recordName);
rMap.put("Primary_Email",email);
newRecord = zoho.crm.createRecord(newModule, rMap);
newRecordId = newRecord.get("id");
info newRecordId;
```
### Get all necessary Related Lists and Modify it to the New Record
The common related lists in records are Contacts, Deals, Notes and Activities (Tasks, Events, Calls). This list may differ/be longer on a case-to-case basis depending on the nature of your modules, but for the purpose of this example, we will focus on the four.

#### 1. Related Contacts
In this example, "Household" and "Business" are both lookup fields on the Contact module. A `for loop` is used to iterate through every Contact.
```javascript
allContacts = zoho.crm.getRelatedRecords("Contacts", oldModule, recordId);
for each c in allContacts
{
	delete = zoho.crm.updateRecord("Contacts", c.get("id"), {"Household" : ""});
	update = zoho.crm.updateRecord("Contacts", c.get("id"), {"Business" : newRecordId});
	info delete;
	info update;
}
```
*Note: Modify the script above accordingly based on your module set up*

#### 2. Related Deals
The same is done for all related deals.
```javascript
allDeals = zoho.crm.getRelatedRecords("Deals", oldModule, recordId);
if (allDeals.size() > 0)
{
  for each d in allDeals
  {
	  delete = zoho.crm.updateRecord("Deals", d.get("id"), {"Household_Name" : ""});
	  update = zoho.crm.updateRecord("Deals", d.get("id"), {"Business_Name" : newRecordId});
	  info delete;
	  info update;
  }
}
```
*Note: Modify the script above accordingly based on your module set up*

#### 3. Related Notes
Related Notes are migrated by copying both the **Note Title** and **Note Content** from the old record, and creating in the new. A `for loop` is used to iterate through every note.

```javascript
allNotes = zoho.crm.getRelatedRecords("Notes", oldModule, recordId);
if(allNotes.size() > 0)
{
	for each  n in allNotes
	{
		notesMap = {"Parent_Id":newRecordId,"Note_Title":n.get("Note_Title"),"Note_Content":n.get("Note_Content"),"$se_module":newModule};
		create = zoho.crm.createRecord("Notes",notesMap);
		info create;
	}
}
```
*Note: The Notes related list has a universal structure so changes to the script are not needed.

#### 4. Open Activities
Get all Activities on the old record, iterate through a `for loop`, then modify the Activity by linking it to the new record. 
* `"$se_module"` determines the **module** the activity should lookup to
* `What_Id` links the activity to the specific **record** lookup
* Since there are 3 different types of activities (Tasks, Events, Calls), `a.get("Activity_Type")` is used as an identifier for the system in `updateRecord`.

```javascript
allAct = zoho.crm.getRelatedRecords("Activities", oldModule, recordId);
if (allAct.size() > 0)
{
for each a in allAct
	{
		idMap = Map();
		idMap.put("id",newRecordId);
		aMap = Map();
		aMap.put("$se_module",newModule);
		aMap.put("What_Id",idMap);
		updateAct = zoho.crm.updateRecord(a.get("Activity_Type"), a.get("id"), aMap);
		info updateAct;
	}
}
```

#### Closed Activities
When activites related to records are closed, they are moved to a different related list in the system called "Activities_History". For traceability, it's in good practice that closed activites are also migrated to the new record.

```javascript
allPastAct = zoho.crm.getRelatedRecords("Activities_History", oldModule, recordId);
if (allPastAct.size() > 0)
{
	for each ap in allPastAct
	{
		idMap = Map();
		idMap.put("id",newRecordId);
		aMap = Map();
		aMap.put("$se_module",newModule);
		aMap.put("What_Id",idMap);
		updatePastAct = zoho.crm.updateRecord(ap.get("Activity_Type"), ap.get("id"), aMap);
		info updatePastAct;
	}
}
```

### Download & Upload Attachments
To leave no stones unturned, all attachment(s) on the old record are also migrated by downloading the attachment(s), and uploading to the new record.
```javascript
attachment = invokeurl
[
	url: "https://www.zohoapis.com/crm/v2/"+oldModule+"/"+recordId+"/Attachments"
	type: GET
	connection: "zohocrm" //Change this to your connection name
];
if (attachment.get("data").size() > 0)
{
	for each atm in attachment.get("data")
	{
		download = invokeurl
		[
			url: "https://www.zohoapis.com/crm/v2/"+oldModule+"/"+recordId+"/Attachments/"+atm.get("id")
			type: GET
			connection: "zohocrm" //Change this to your connection name
		];
		
		upload = zoho.crm.attachFile(newModule,newRecordId,download);
		info upload;
	}
}
```
### Delete Old Record
When the transition is completed, delete the old record from the original module.
```javascript
delete = invokeurl
[
	url: "https://www.zohoapis.com/crm/v2/"+oldModule+"?ids="+recordId+"&wf_trigger=true"
	type: DELETE
	connection: "zohocrm" //Change this to your connection name
];
info delete;
```

## Example Use Case - Convert Contacts back to Leads
The ability to convert Contacts back to Leads is sought after by many. Since Zoho does not have that functionality yet, we're gonna build it using the scripts above.

### What happens during a Lead Conversion?
When a Lead is converted into a Contact, a few other things happen:
* An Account is created
* A new Deal is created (optional)
To undo a conversion, we need to reverse engineer the process.

### What we need to do
In order to convert a Contact back to a Lead, here's what we need the script to do:
* Get all necessary fields and create the Lead record (make sure all mandatory fields are mapped).
* Copy over all Related Notes
* Reassign all related Activties (past and present)
	* "Who_Id" has to be set to "" to remove the connection with the Contact (this is necessary to prevent the activities from being deleted when the Contact is deleted later).
* Copy over all Attachments (make sure that the attachments are moved to the Contact during the Zoho conversion. If it was moved to the Deal, you will need to reconfigure the script).
* Delete all Related Deals
* Delete the Related Account
* Delete the Contact Record

### Create a Button for the Function
For best user experience, create a [custom button](https://help.zoho.com/portal/en/kb/crm/customize-crm-account/custom-links-and-buttons/articles/custom-buttons) that says "Undo Convert" on the "View Page" which will execute the script below.
* Add a note in the button description to warn users that the related Account and Deals will be deleted.
* Add a success message with the URL of the Lead at the end of the script to notify users that the "Undo Conversion" is complete.

```javascript
//Change below to the relevant old & new modules
oldModule = "Contacts";
newModule = "Leads";
oldRecord = zoho.crm.getRecordById(oldModule,recordId);

//Get all Necessary Fields and Add to a Map - add/modify as needed
rMap = Map();
rMap.put("First_Name",oldRecord.get("First_Name"));
rMap.put("Last_Name",oldRecord.get("Last_Name"));
rMap.put("Email",oldRecord.get("Email"));
rMap.put("Company",ifNull(oldRecord.get("Account_Name"),{"name":""}).get("name"));

//Create Record
newRecord = zoho.crm.createRecord(newModule,rMap);
newRecordId = newRecord.get("id");
info newRecordId;

//Copy Related Notes
allNotes = zoho.crm.getRelatedRecords("Notes",oldModule,recordId);
if(allNotes.size() > 0)
{
	for each  n in allNotes
	{
		notesMap = {"Parent_Id":newRecordId,"Note_Title":n.get("Note_Title"),"Note_Content":n.get("Note_Content"),"$se_module":newModule};
		create = zoho.crm.createRecord("Notes",notesMap);
		info create;
	}
}

//Get and Modify Related Activities
allAct = zoho.crm.getRelatedRecords("Activities",oldModule,recordId);
if(allAct.size() > 0)
{
	for each  a in allAct
	{
		idMap = Map();
		idMap.put("id",newRecordId);
		aMap = Map();
		aMap.put("$se_module",newModule);
		aMap.put("What_Id",idMap);
		aMap.put("Who_Id","");
		updateAct = zoho.crm.updateRecord(a.get("Activity_Type"),a.get("id"),aMap);
		info "UpdateACT : " + updateAct;
	}
}

//Get and Modify Related Closed Activities
allPastAct = zoho.crm.getRelatedRecords("Activities_History",oldModule,recordId);
if(allPastAct.size() > 0)
{
	for each  ap in allPastAct
	{
		idMap = Map();
		idMap.put("id",newRecordId);
		aMap = Map();
		aMap.put("$se_module",newModule);
		aMap.put("What_Id",idMap);
		aMap.put("Who_Id","");
		updatePastAct = zoho.crm.updateRecord(ap.get("Activity_Type"),ap.get("id"),aMap);
		info "UPDATE PAST ACT : " + updatePastAct;
	}
}

//Download & Upload Attachments
attachment = invokeurl
[
	url :"https://www.zohoapis.com/crm/v2/" + oldModule + "/" + recordId + "/Attachments"
	type :GET
	connection:"zohocrm"
];
if(attachment.get("data").size() > 0)
{
	for each  atm in attachment.get("data")
	{
		download = invokeurl
		[
			url :"https://www.zohoapis.com/crm/v2/" + oldModule + "/" + recordId + "/Attachments/" + atm.get("id")
			type :GET
			connection:"zohocrm"
		];
		upload = zoho.crm.attachFile(newModule,newRecordId,download);
		info upload;
	}
}

//Delete Related Deals
allDeals = zoho.crm.getRelatedRecords("Deals",oldModule,recordId);
if(allDeals.size() > 0)
{
	for each  d in allDeals
	{
		delete = invokeurl
		[
			url :"https://www.zohoapis.com/crm/v2/Deals?ids=" + d.get("id") + "&wf_trigger=true"
			type :DELETE
			connection:"zohocrm"
		];
		info delete;
	}
}

//Deleted Related Account
accountid = ifNull(oldRecord.get("Account_Name"),{"id":"none"}).get("id");
if(accountid != "none")
{
	delete = invokeurl
	[
		url :"https://www.zohoapis.com/crm/v2/Accounts?ids=" + accountid + "&wf_trigger=true"
		type :DELETE
		connection:"zohocrm"
	];
	info delete;
}

//Delete the Old Record
delete = invokeurl
[
	url :"https://www.zohoapis.com/crm/v2/" + oldModule + "?ids=" + recordId + "&wf_trigger=true"
	type :DELETE
	connection:"zohocrm"
];
info delete;

//Construct success msg
newURL = "https://crm.zoho.com/crm/org707073965/tab/Leads/" + newRecordId;
return "This Contact has been converted back into a Lead here: " + newURL;
```
