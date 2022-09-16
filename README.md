# SecretsnCertsExpirationNotification
Using Power Automate to proactively notify the Azure Active Directory administrators of upcoming client secret and certificate expirations.
Here’s how I solved it using Power Automate:

Create (or use an existing) Azure AD app registration that has ONE of the following Application Permissions (starting from the least and ending with the most restrictive option) - Application.Read.All, Application.ReadWrite.All, Directory.Read.All, or Directory.AccessAsUser.All.
Create a Scheduled Flow to run daily or weekly depending on how often you want to be alerted.
Initialize variable (String) – appId – this is the appID of the application.
Initialize variable (String) – displayName – this will be used for the display name of the application.
Initialize variable (String) – clientSecret – this needs to be set with the client secret of the Azure AD application created or chosen in step 1.
Initialize variable (String) – clientId – this needs to be set with the application (client) ID of the Azure AD application created or chosen in step 1.
Initialize variable (String) – tenantId – this needs to be set with the tenant ID of the Azure AD application created or chosen in step 1.
Initialize variable (Array) – passwordCredentials – this variable will be used to populate the client secrets of each Azure AD application.
Initialize variable (Array) – keyCredentials – this variable will be used to populate the certificate properties of each Azure AD application.
Initialize variable (String) – styles – this is some CSS styling to highlight Azure AD app secrets and expirations that are going to expire in 30 days (yellow) vs 15 days (red). You can adjust these values accordingly to meet your needs.
Content of this step: { "tableStyle": "style="border-collapse: collapse;"", "headerStyle": "style="font-family: Helvetica; padding: 5px; border: 1px solid black;"", "cellStyle": "style="font-family: Calibri; padding: 5px; border: 1px solid black;"", "redStyle": "style="background-color:red; font-family: Calibri; padding: 5px; border: 1px solid black;"", "yellowStyle": "style="background-color:yellow; font-family: Calibri; padding: 5px; border: 1px solid black;"" }

Initialize variable (String) – html – this creates the table headings and rows that will be populated with each of the Azure AD applications and associated expiration info.
Content of this step:

Initialize variable (Float) – daysTilExpiration – this is the number of days prior to client secret or certificate expiration to use in order to be included in the report

We need to request an authentication token using our tenantId, clientId, and clientSecret variables.

The Parse JSON step will parse all the properties in the returned token request. The JSON schema to use is as follows: {     "type": "object",     "properties": {         "token_type": {             "type": "string"         },         "expires_in": {             "type": "integer"         },         "ext_expires_in": {             "type": "integer"         },         "access_token": {             "type": "string"         }     } }

Initialize variable (String) – NextLink – This is the graph API URI to request the list of Azure AD applications. The $select only returns the appId, DisplayName, passwordCredentials, and keyCredentials, and since graph API calls are limited to 100 rows at a time, I bumped my $top up to 999 so it would use less API requests (1 per 1000 apps vs 10 per 1000 apps). https://graph.microsoft.com/v1.0/applications?$select=appId,displayName,passwordCredentials,keyCredentials&$top=999

Next, we enter the Do until loop. It will perform the loop until the NextLink variable is empty. The NextLink variable will hold the @odata.nextlink property returned by the API call. When the API call retrieves all the applications in existence, there is no @odata.nextlink property. If there are more applications to retrieve, the @odata.nextlink property will store a URL containing the link to the next page of applications to retrieve. The way to accomplish this is to click “Edit in advanced mode” and paste @empty(variables('NextLink')).

The next step in the Do until loop uses the HTTP action to retrieve the Azure AD applications list. The first call will use the URL we populated this variable within step 15.

A Parse JSON step is added to parse the properties from the returned body from the API call.

The content of this Parse JSON step is as follows:

{ "type": "object", "properties": { "@@odata.context": { "type": "string" }, "value": { "type": "array", "items": { "type": "object", "properties": { "appId": { "type": "string" }, "displayName": { "type": "string" }, "passwordCredentials": { "type": "array", "items": { "type": "object", "properties": { "customKeyIdentifier": {}, "displayName": {}, "endDateTime": {}, "hint": {}, "keyId": {}, "secretText": {}, "startDateTime": {} }, "required": [] } }, "keyCredentials": { "type": "array", "items": { "type": "object", "properties": { "customKeyIdentifier": {}, "displayName": {}, "endDateTime": {}, "key": {}, "keyId": {}, "startDateTime": {}, "type": {}, "usage": {} }, "required": [] } } }, "required": [] } }, "@@odata.nextLink": { "type": "string" } } }

A Get future time action will get a date in the future based on the number of days you’d like to start receiving notifications prior to expiration of the client secrets and certificates.

Next a foreach – apps loop will use the value array returned from the Parse JSON step of the API call to take several actions on each Azure AD application.

Set variable (String) – appId – uses the appId variable we initialized in step 3 to populate it with the application ID of the current application being processed.

Set variable (String) – displayName – uses the displayName variable we initialized in step 4 to populate it with the displayName of the application being processed.

Set variable (String) – passwordCredentials – uses the passwordCredentials variable we initialized in step 5 to populate it with the client secret of the application being processed.

Set variable (String) – keyCredentials – uses the keyCredentials variable we initialized in step 5 to populate it with the client secret of the application being processed.

A foreach will be used to loop through each of the client secrets within the current Azure AD application being processed.

The output from the previous steps to use for the foreach input is the passwordCreds variable.

A condition step is used to determine if the Future time from the Get future time step 19 is greater than the endDateTime value from the current application being evaluated.

If the future time isn’t greater than the endDateTime, we leave this foreach and go to the next one.

If the future time is greater than the endDateTime, we first convert the endDateTime to ticks. Ticks is a 100-nanosecond interval since January 1, 0001 12:00 AM midnight in the Gregorian calendar up to the date value parameter passed in as a string format. This makes it easy to compare two dates, which is accomplished using the expression ticks(item()?[‘endDateTime’]).

Next, use a Compose step to convert the startDateTime variable of the current time to ticks, which equates to ticks(utcnow()).

Next, use another Compose step to calculate the difference between the two ticks values, and re-calculate it using the following expression to determine the number of days between the two dates.

div(div(div(mul(sub(outputs('EndTimeTickValue'),outputs('StartTimeTickValue')),100),1000000000) , 3600), 24)

Set the variable daystilexpiration to the output of the previous calculation.
Set variable (String) – html – creates the HTML table. The content of this step is as follows:
Another foreach will be used to loop through each of the certificates within the current Azure AD application being processed. This is a duplication of steps 25 through 33 except that it uses the keyCredentials as its input, compares the future date against the currently processed certificate endDateTime, and the Set variable – html step is as follows:
Immediately following the foreach – apps loop, as a final step in the Do while loop is a Set NextLink variable which will store the dynamic @odata.nextlink URL parsed from the JSON of the API call.

Append to variable (Array) – html – Immediately following the Do while loop ends, we close out the html body and table by appending

Application ID	Display Name	Days until Expiration	Type	Expiration Date
@{variables('appId')}	@{variables('displayName')}	@{variables('daystilexpiration')}	Secret	@{formatDateTime(item()?['endDateTime'],'g')}
@{variables('appId')}	@{variables('displayName')}	@{variables('daystilexpiration')}	Certificate	@{formatDateTime(item()?['endDateTime'], 'g')}
to the variable named html.
Finally, send the HTML in a Send an e-mail action, using the variable html for the body of the e-mail.
