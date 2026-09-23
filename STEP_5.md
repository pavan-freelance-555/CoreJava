That is the one remaining fact we have not yet proved.

Spring 6 worked in DEV and UAT for one of two main reasons:

Possibility 1: Spring 6 sent different SOAPAction values

Your SOAPAction is populated from:

${brkmidoffice.endpoint}

That property normally changes by environment.

For example:

DEV:
brkmidoffice.endpoint=https://dev.../BrkMidOfficeService

UAT:
brkmidoffice.endpoint=https://uat.../BrkMidOfficeService

Preprod:
brkmidoffice.endpoint=https://wealth-uat.../BrkMidOfficeService

Because the endpoint is also incorrectly used as soapActionUri, Spring 6 sends a different SOAPAction in every environment.

The DEV and UAT values may happen to match what their SOAP servers accept, while the preprod value does not.

Possibility 2: DEV/UAT SOAP servers are more tolerant

Spring 6 may send the same type of incorrect action in every environment:

SOAPAction = endpoint URL

The DEV/UAT servers may ignore it and identify the operation from the SOAP body:

<soap:Body>
    <GetMOTUserProfile>
        ...
    </GetMOTUserProfile>
</soap:Body>

Preprod does not ignore it. It attempts to find an operation corresponding to the received SOAPAction and fails.

Important rule

If all these things were identical:

Same Spring 6 build
Same resolved properties
Same outgoing request
Same gateway
Same SOAP-server version
Same WSDL and server configuration

then DEV, UAT and preprod would behave identically.

Because the results differ, at least one of them is different.

How to prove which possibility it is

Enable Spring 6 JAX-WS transport logging in all three environments:

-Dcom.sun.xml.ws.transport.http.client.HttpTransportPipe.dump=true

Perform the same login once in each environment.

Capture only these lines:

POST ...
Content-Type: ...
SOAPAction: "..."

Build this table:

Environment	Endpoint	Actual SOAPAction	Result

DEV	?	?	Success
UAT	?	?	Success
Preprod	wealth-uat...	Endpoint URL	Failure


Result A: SOAPActions differ

For example:

DEV     → SOAPAction: "GetMOTUserProfile"
UAT     → SOAPAction: "GetMOTUserProfile"
Preprod → SOAPAction: "https://.../BrkMidOfficeService"

Conclusion:

> Preprod loads a different property/configuration value.



Result B: all send endpoint URLs

For example:

DEV     → SOAPAction: "https://dev.../BrkMidOfficeService"
UAT     → SOAPAction: "https://uat.../BrkMidOfficeService"
Preprod → SOAPAction: "https://preprod.../BrkMidOfficeService"

DEV/UAT succeed, but preprod fails.

Conclusion:

> The backend service or gateway handling the preprod request is configured differently or has a different SOAP runtime/deployment.



One important clue from your screenshot

The failing request contains:

https://wealth-uat.../BrkMidOfficeService

But you said the failure happens from preprod.

Therefore, first verify:

Why is the preprod application using a URL containing “uat”?

It might be correct naming in your organization, but it could also mean preprod loaded the wrong property file or is calling the wrong backend.

The honest conclusion is:

> We know exactly why preprod failed: it received the endpoint URL as SOAPAction instead of GetMOTUserProfile. We cannot yet state why DEV/UAT accepted it until we capture their actual SOAPAction headers and compare the resolved endpoints.