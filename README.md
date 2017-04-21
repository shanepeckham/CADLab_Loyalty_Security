# CADLab_Loyalty_Security

This lab demonstrates how to secure the Loyalty Scenario lab which should be completed before this lab is attempted, it can be found here [Loyalty](https://github.com/shanepeckham/CADLab_Loyalty)

# Securing our environment

In this lab we will secure the environment that we provisioned in the scenario lab [Loyalty](https://github.com/shanepeckham/CADLab_Loyalty) . We will be applying security to this environment utilising a combination of the following:

* The API Management subscription keys to control authorisation to the APIs
* Network security groups to close ports in IaaS
* Azure Active Directory to handle authentication
* Azure Function API Authentication keys
* IP restriction to limit access to App Service APIs and Logic Apps

## Adding Authentication to the API App

Navigate to the App Service API App, its default name will be CADAPIMasterSite[hash] and click on the Authentication / Authorization blade, and select the value 'On' for the App Service Authentication section, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/APIsecure1.png)

Select "Log in with Azure Active Directory" in the section *Action to take when request is not authenticated*

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/APIsecure2.png)

Now select Azure Active Directory and value "Express Mode" in the *Management Mode* section. We are now registering the API app within the default directory of the logged in Azure subscription, keep the value "Create new App", see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/APIsecure3.png)

Click Ok, and Click Save back on the Authentication / Authorization blade. Note, you will no longer be able to test this API in the API Management Developer Portal unless you have a valid JWT token.

## Adding Authentication headers to the Logic App to pass to the App Service API 

We now want to test our API App and ensure we can call it successfully via the Logic app. We now need to get the properties of our Default Active Directory store so that we can successfully Authenticate.

We need to generate JSON headers that look like the example below, and we need to find the values to populate this object:
```
{
    "audience":"",
    "clientId":"",
    "secret":"",
    "tenant":"",
    "type":"ActiveDirectoryOAuth"
}
```
Navigate to Azure Active Directory within the main menu within the portal and select "Default Directory". Copy the value in *Directory ID*, this will be mapped to the name field *tenant*, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/ad1.png)

Now click on the menu option "App Registrations" and select your API App, this will open the detail view for the registered app, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/ad2.png)

We want to copy the value of the field *Application ID* which will be mapped to the name field *audience* and *clientId*, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/ad3.png)

Now we need to get the final value, *secret*. Click on the menu option 'Keys' and enter a value in *Description* and select *Expires* "In 1 year", see below: 

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/ad4.png)

Click Save, this will generate a new secret that we can copy and use to populate the value in the field *secret*. Here is an example of a completed Authentication header JSON:
```
{
    "audience":"d983cbac-8d8b-4a33-ab9e-b495ff903975",
    "clientId":"d983cbac-8d8b-4a33-ab9e-b495ff903975",
    "secret":"[your generated secret]",
    "tenant":"5e337db4-14b1-4a41-a1f2-c55f31e882b8",
    "type":"ActiveDirectoryOAuth"
}
```

We now need to pass these values in with our original request, so let's go and add them to our initial schema. I will use https://jsonschema.net to generate mine. Here is my schema:
```
{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "definitions": {},
    "id": "http://example.com/example.json",
    "properties": {
        "APIMkey": {
            "id": "/properties/APIMkey",
            "type": "string"
        },
        "audience": {
            "id": "/properties/audience",
            "type": "string"
        },
        "clientId": {
            "id": "/properties/clientId",
            "type": "string"
        },
        "id": {
            "id": "/properties/id",
            "type": "integer"
        },
        "secret": {
            "id": "/properties/secret",
            "type": "string"
        },
        "tenant": {
            "id": "/properties/tenant",
            "type": "string"
        },
        "type": {
            "id": "/properties/type",
            "type": "string"
        }
    },
    "type": "object"
}
```
Now we can use the Parse JSON step in our Logic App to handle the incoming body for us so that we can easily access the fields we need that are specific for authentication, add it between your original request step and the API App Contact query step see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/logicsecurity1.png)

Paste your new schema that you just generated into this field. The parse JSON will provide us with a nicely formatted output that we can plug into the Authorisation headers for the Lgeacy API app (Mine is called Contacts GetById 2 in the example below), see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/logicsecurity3.png)

Call your Logic App via Postman, passing the values in the Body and you should successfully Authenticate. If you get a 302, then it means that you are allowing HTTP requests in your API App definition in API Management, this should be set to HTTPS only, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/logicsecurity4.png)

Note, do not set this for the Legacy API, only the APP service API.

This is what my postman request Body looks like:
```
{ 
"id" : "1",
"APIMKey": "8ffba9fbb9784a57bccc68502cc341ae",
"audience":"d983cbac-8d8b-4a33-ab9e-b495ff903975",
"clientId":"d983cbac-8d8b-4a33-ab9e-b495ff903975",
"secret":"[My secret]",
"tenant":"5e337db4-14b1-4a41-a1f2-c55f31e882b8",
"type":"ActiveDirectoryOAuth"
}
```

## IP Restricting the App Service API App to be callable only from API Management

Now that we have authentication in place on the API app, we can add a further level of access control by restricting the IP that can call the App Service App to our API Management instance. 

To get the API Management IP navigate to API Management instance provisioned and go to the *Overview* menu item, copy the value(s) from the Public virtual IP (VIP) addresses, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/apimip.png)

Now navigate to the App Service API App, its default name will be CADAPIMasterSite[hash] and click on the *Resource Explorer* menu item, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource1.png)

This will open a largely empty pane with a button link *Go*, click this and a new browser window will open which will display the Azure Resource representation of the App Service API, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource2.png)

Now expand the following sections under the selected and highlighted site, namely --> Config --> Web, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource3.png)

Now click the edit button at the stop of the screen, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource4.png)

Scroll all the way to the bottom to the entry ```   "ipSecurityRestrictions": null, ```, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource5.png)

Change this to the following

```
"ipSecurityRestrictions": [
      {
        "ipAddress": "[APIM IP Address]",
        "subnetMask": "255.255.0.0"
      }
    ],
```

Now scroll back to the top, select *Read/Write* mode and click the Patch button, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource7.png)

Now navigate back to the Overview menu item of the App Service API in the Azure portal and restart the app, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/resource8.png)

If you now click the *Browse* button you should not be able to access the swagger harness as you did in Step 2 of the scenario lab [Loyalty] . If you try and access the App Service API you will get a **HTTP 403 Forbidden**, this means you have successfully IP restricted the app to be only invocable by our API Management instance.

To test this call the Logic app again and you should still have a successfully function API Management step.


## Adding an authentication key to the serverless Azure Function

At the time of writing the Authentication / Authorisation feature of App Services does not work on Azure Functions, so we will use the API Definition functionality to both generate us an Authentication key and to place the Azure function behind the API Management Gateway so that is requires both an API Management Authorisation key and an Azure Function Authentication key.

Navigate to the serverless function, select the *API Definition* section and click the Function button under *API Definition Source*. We are now going to auto-generate a swagger definition for our Function which we will use to expose the Function within API Management, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec1.png)

Now click the button *Generate API Definition Template*, this will populate the swagger json definition for this function, then cick Save, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec2.png)

Now on the blade to the left expand the section *API Definition Key* and copy this value for future reference, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec3.png)

This is our authorisation key which will be required to invoke this function from now one, without it the user will get a **401 Unauthorised**

Now copy the value in the section *API Definition URL* and store it for future reference, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec4.png)

### Adding the Azure serverless function to API Management

We will now add the function to API Management so that it can only be invoked with an APIM Authorisation or subscription key. Navigate to the API Management instance and navigate to the Publisher portal, select 

Click on the overview blade and select 'Publisher Portal' (note we will use the old portal while the new portal is still in preview), see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/apimpublisherportal.png)

This will navigate you to the Publisher portal, select Import API, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/importAPI.png)

Now enter the following values:

* Select "From Url"
* Paste your API Definition URL for your function from the step above
* Select "Swagger" as the specification format
* Select New API
* Type "GenerateCoupon" into the Web API URL Suffix field
* Click Products and select "Unlimited"
* Click Save

The function should now import into API Management, now we want to add the Authorisation key as a Query Parameter as this is how it needs to be passed to the Function. Click the *Operations* section and the Post operation, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec5.png)

Now in the *Request* section select "Add Query Parameter". Add the value code in the "Name" field and click Save, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec6.png)

Lastly, navigate back to the *Settings* section and copy the value in the "This is what the Web API URL is going to look like:" field + the value in from your function, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcapimurl.png)

# Add the GenerateCoupon API Management fronted Serverless function to the Logic App

Now we want to remove the Azure Function step we added in the the scenario lab [Loyalty] as, at the time of writing, we will not be able to pass all of the parameters required using either an Azure Function or API Management connector within a Logic App, instead we will replace it with the HTTP step, see below:

Remove this -> ![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec7.png)

And replace with this -> ![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec8.png) Connector type HTTP with Trigger *HTTP - HTTP*

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec9.png) 

Now populate the values:

***Method: POST***
***Uri: This is the value that you copied above in the *"This is what the Web API URL is going to look like:"* + "?code=" + the value from the *API Definition Key* above. For example, this is what mine looks like:***

```
https://cadapimldwy3qi7smdni.azure-api.net/GenerateCoupon/api/GenerateCoupon?code=7au/JKrmVQiqn2byK84DWTJpl7W46jugbqY43pe2oUWxdkOrzH==
```

In the Headers section add values just like you would do calling APIM from Postman, e.g.

***Content-Type: application/json***
***Ocp-Apim-Subscription-Key***: Here you can use your Logic app dynamic value slug
***Ocp-Apim-Trace: true*** (to enable tracing)

And in the body section:
```
{
  "name": "@body('QueryContactsById')[0]['name']"
}
```

See below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/funcsec11.png) 

You should now be able to call your function from within the Logic app securely, we will use the Uri value statically for now but it could be dynamically exposed and the Function Authentication key could be passed in dynamically by the Logic app calling process.

# Restrict the IPs that can call the Logic app

So far we have controlled access to all components except the Logic app, we will now add IP restriction to the Logic app. Navigate to the *Access control configuration* section of your logic app, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/logicsec1.png) 

Here you can enter IP ranges for the clients/servers that are allowed to invoke this Logic App. 
Select the "Specific IP Ranges" in the section *Allowed inbound IP addresses* and enter a range, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/logicsec2.png) 

I have entered an IP address different to my machine's address to see if I can still invoke the Logic App from Postman, I can't, I get an error, see below:

![alt text](https://github.com/shanepeckham/CADLab_Loyalty/blob/master/Images/logicsec3.png) 




























