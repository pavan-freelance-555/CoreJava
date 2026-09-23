Yes, different environment URLs normally point to different SOAP server deployments:

DEV URL     → DEV SOAP server
UAT URL     → UAT SOAP server
Preprod URL → Preprod SOAP server

But different URLs should not normally require different SOAPActions when they expose the same WSDL contract.

URL versus SOAPAction

Think of it this way:

Service URL = Building address
SOAPAction  = Work you want performed

For example:

DEV address:
https://dev.../BrkMidOfficeService

UAT address:
https://uat.../BrkMidOfficeService

Preprod address:
https://preprod.../BrkMidOfficeService

But the requested work should remain the same:

SOAPAction: "GetMOTUserProfile"

So the correct design is:

Environment	Service URL	SOAPAction

DEV	DEV URL	GetMOTUserProfile
UAT	UAT URL	GetMOTUserProfile
Preprod	Preprod URL	GetMOTUserProfile


Only the destination URL changes. The operation action should come from the WSDL.

What your current configuration does

Your configuration connects both values:

endpointAddress = ${brkmidoffice.endpoint}
soapActionUri   = ${brkmidoffice.endpoint}

Therefore, it may send:

Environment	Endpoint	Incorrect SOAPAction

DEV	DEV URL	DEV URL
UAT	UAT URL	UAT URL
Preprod	Preprod URL	Preprod URL


That is incorrect because the SOAPAction should be:

GetMOTUserProfile

Why could DEV/UAT still accept it?

Different URLs can route to different infrastructure:

DEV URL
   ↓
DEV load balancer
   ↓
DEV SOAP server

Preprod URL
   ↓
Preprod load balancer/gateway
   ↓
Preprod SOAP server

The servers might have different behavior.

For example, the DEV server might receive:

SOAPAction: "https://dev.../BrkMidOfficeService"

and decide:

SOAPAction does not match
        ↓
Look inside SOAP body
        ↓
Body contains GetMOTUserProfile
        ↓
Execute GetMOTUserProfile

Preprod might receive:

SOAPAction: "https://preprod.../BrkMidOfficeService"

and decide:

SOAPAction does not match
        ↓
Reject immediately
        ↓
Return SOAP Fault

This difference could come from:

Different SOAP server/runtime versions

Different deployed service builds

Different gateway policies

Different WSDL versions on the server

Different endpoint routing

Different validation settings


But this is not proven yet

We know only this for certain:

Preprod received endpoint URL as SOAPAction
WSDL expects GetMOTUserProfile
Preprod rejected it

We do not yet know what DEV/UAT actually received.

They may be accepting the wrong action, or their Spring 6 applications may be sending a different/correct action because different configuration is loaded.

The one comparison that proves it

Capture the SOAPAction header from Spring 6 in DEV and UAT.

If you find:

DEV     SOAPAction = DEV endpoint URL       → works
UAT     SOAPAction = UAT endpoint URL       → works
Preprod SOAPAction = Preprod endpoint URL   → fails

then yes:

> The different server deployments are handling the incorrect SOAPAction differently.



But if you find:

DEV     SOAPAction = GetMOTUserProfile      → works
UAT     SOAPAction = GetMOTUserProfile      → works
Preprod SOAPAction = endpoint URL           → fails

then:

> The servers are not behaving differently; preprod is loading different client configuration.



So, different URLs can lead to servers with different validation behavior, but the URL itself does not decide the SOAPAction. The correct SOAPAction should remain GetMOTUserProfile in every environment.