# Teams Requests
Custom Microsoft Teams creation process using SharePoint Online and Microsoft Flow that allows for additional metadata to be collected during the request, enabling organizations to implement their governance requirements while still allowing automated self-service.

Employees will fill out a request form in SharePoint Online (or in PowerApps, should you choose to create one). A Flow will trigger on that new SharePoint Online item and use the Microsoft Graph API to create a new team based on the selected template, assign the requested owners, and notify the requester with a Teams notification.

![Process Diagram](Images/ProcessDiagram.png)

## Steps
1. [Register an Application in Azure Active Directory](#register-an-application-in-azure-active-directory)
2. [Create SharePoint Online Custom Lists](#create-sharepoint-online-lists)
3. [Create a Power Automate flow](#create-power-automate-flow) (Under construction)
4. or [Import the flow](#or-import-the-power-automate-flow)
5. [Create a PowerApp](#create-powerapp)
6. [Pin the app to the sidebar in Microsoft Teams](#publish-powerapp-to-teams-and-pin)

## Register an Application in Azure Active Directory
1. Browse to https://aad.portal.azure.com
2. Register a new application: https://portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationsListBlade
![AAD app registration](Images/AAD-AppReg.jpg)
3. Copy the `Application (client) ID` and the `Directory (tenant) ID` from the Overview page into OneNote/notepad
4. Click on `Certificates & secrets` and create a new client secret (copy/paste it into OneNote/notepad)
![AAD secret](Images/AAD-secret.jpg)
5. Click on `API permissions` and add the following permissions for Microsoft Graph (Application, not delegated): **Group.ReadWrite.All**, **User.ReadWrite.All**, **Directory.ReadWrite.All**
![AAD graph](Images/AAD-Graph.jpg)
![AAD permissions](Images/AAD-Permissions.jpg)
6. Click the `Grant admin consent for [tenant]` button

## Create SharePoint Online lists
The following custom lists in SharePoint are used in the sample Flow and can be modified/omitted to fit your needs. Any alterations may require updates to get the Flow to work, depending on the change.
### Teams Templates
Create teams in Microsoft Teams to act as your templates, then store references to those templates in this simple custom list to be chosen from when someone requests a new Team. This list will be used as a lookup on the Teams Requests list. You can copy/paste the TeamID from the Teams Admin Center.
![TeamsTemplates list](Images/List-TeamsTemplates.gif)
![TeamsTemplates list settings](Images/List-TeamsTemplates-Settings.gif)
### Business Units
This custom list (with no additional custom fields) is entrirely optional and just an example of the type of extra information you can collect and store to meet your governance needs. This list will be used as a lookup on the Teams Requests list. 
![Business Units list](Images/List-BusinessUnits.gif)
### Teams Requests
This custom list is where people will submit their requests. You can certainly make a PowerApp on top of this list for people to use.
![Teams Requests list](Images/List-TeamsRequests.gif)
![Teams Requests list settings](Images/List-TeamsRequests-Settings.gif)

## Create Power Automate flow
*Or you can import the flow in the [step after this](#or-import-the-power-automate-flow) and update the various actions*

1. Browse to https://flow.microsoft.com

2. Click on `Create` in the left menu

3. Start from blank: `Automated flow`

4. Give your flow a name and choose your flow's trigger: `When an item is created (SharePoint)`, click `Create`

![Create flow](Images/Flow-0-Create.png)

5. Select the Site where you created the lists above and select the List Name to: `Teams Requests`

![Flow: trigger](Images/Flow-1-trigger.png)

6. Add 3 new steps using the `Initialize variable [Variables]` action for each, setting the Values to the IDs you copied when you created your AAD App Registration above

![Flow: set ID variables](Images/Flow-2-IDs.png)

7. Add 3 new steps using the `Initialize variable [Variables]` action for each, setting AllowGuests to the expression:
```
if(triggerBody()?['AllowGuests'], 'True', 'False')
```

![Flow: set variables](Images/Flow-2-Variables.png)

8. Add a new step using the `HTTP [HTTP]` action
- The is the first of several HTTP calls. This one will get the bearer token we need to call the Graph later.

Method: POST

URI: ```https://login.microsoftonline.com/@{variables('TenantId')}/oauth2/token```

Body:
```
client_id=@{variables('ClientId')}&
client_secret=@{variables('SecretId')}&
grant_type=client_credentials&
resource=https%3A%2F%2Fgraph.microsoft.com
```

![Flow: HTTP action for bearer token](Images/Flow-3-HTTP.png)

9. Add a new step using the `Parse JSON [Data Operations]` action, seeting the Content to the HTTP action `Body` and the following schema:

Content: ```@{body('HTTP')}```

Schema: 
```
{
    "type": "object",
    "properties": {
        "token_type": {
            "type": "string"
        },
        "expires_in": {
            "type": "string"
        },
        "ext_expires_in": {
            "type": "string"
        },
        "expires_on": {
            "type": "string"
        },
        "not_before": {
            "type": "string"
        },
        "resource": {
            "type": "string"
        },
        "access_token": {
            "type": "string"
        }
    }
}
```

![Flow: parse HTTP action Body](Images/Flow-4-Parse.png)

10. Add 2 new steps using the `Get user profile (V2) [Office 365 Users]` action to get the Owner and Secondary Owner profiles using the trigger `Owner Email` and `SecondaryOwner Email` fields

![Flow: get owners](Images/Flow-5-Owners.png)

11. Add a new step using the `Get item [SharePoint]` action, selecting the `Teams Templates` list and the `Template Id` from the SharePoint trigger

![Flow: get template](Images/Flow-6-Template.png)

12. Add a new step using the `HTTP [HTTP]` action (notice the space in between Bearer and the access token in the Headers)

Method: Post

URI: ```https://graph.microsoft.com/v1.0/teams/@{body('Get_Team_Template')['TeamId']}/clone```

Headers: Authorization: ```Bearer @{body('Parse_JSON')?['access_token']}```

Body:
```
{
  "displayName": "@{triggerBody()?['Title']}",
  "description": "@{triggerBody()?['Description']}",
  "partsToClone": "apps,tabs,settings,channels",
  "visibility": "@{toLower(triggerBody()?['Visibility']?['Value'])}",
  "mailNickname": "@{triggerBody()?['Title']}"
}
```

![Flow: clone team](Images/Flow-7-Clone.png)

TODO: finish these instructions (I'm working on it)

13. 


## Or import the Power Automate Flow
Follow this step if you want to import the flow and fix it up rather than create the flow from scratch above.

1. Browse to https://flow.microsoft.com
2. Click on `My flows`
3. Click `Import`
![Flow import](Images/Flow-Import.jpg)
4. Download [TeamsCreationGovernance.zip](/TeamsCreationGovernance.zip) and upload your copy of this file
5. After importing, edit the Flow to plug in your IDs for the AAD Application, your SharePoint Online site and lists, etc.
![Flow IDs](Images/Flow-IDs.jpg)
6. Alter this Flow to meet your needs, including populating the Renewal field, setting the External Sharing attribute for the Team, etc.

## Create PowerApp
![PowerApp: New Team](Images/PowerApp-NewTeam.png)
1. Download [New Team.msapp](/New%20Team.msapp)
2. Browse to https://make.powerapps.com
3. Click on `New app` > `Canvas`
4. Click `Open` and browse to your downloaded .msapp file
5. Once the app opens, click `View` > `Data sources`
6. Delete the `Teams Requests` SharePoint connection
7. Search for `SharePoint` and add a connection to your SharePoint `Teams Requests` list (makie sure it is named the same)
8. Test it to make sure it is working
9. Save and Publish it

## Publish PowerApp to Teams and Pin
1. Add your PowerApp to Teams: https://docs.microsoft.com/en-us/powerapps/maker/canvas-apps/embed-teams-app
2. Publish it in your tenant's app catalog: https://docs.microsoft.com/en-us/microsoftteams/tenant-apps-catalog-teams
3. Pin your app to the sidebar in Teams: https://docs.microsoft.com/en-us/microsoftteams/teams-app-setup-policies