Excellent. Step 1 is complete, and we now have concrete evidence.

Step 1 result

The WSDL says:

<wsdl:operation name="GetMOTUserProfile">
    <soap:operation soapAction="GetMOTUserProfile"/>
</wsdl:operation>

Therefore, the server expects:

SOAPAction: "GetMOTUserProfile"

But the error shows that Spring 6 sent:

SOAPAction: "https://wealth-uat.../BrkMidOfficeService"

So the mismatch is proven:

Value	SOAPAction

WSDL expects	GetMOTUserProfile
Spring 6 sent	Service endpoint URL
Result	SOAP Fault


Step 2: Find why Spring 6 sends the URL

Open your Spring 6 custom JAX-WS bean class.

Search for any of these words:

soapActionUri
SOAPACTION_URI_PROPERTY
SOAPACTION_USE_PROPERTY
ENDPOINT_ADDRESS_PROPERTY
setSoapAction

In IntelliJ or Eclipse:

Ctrl + Shift + F

Search first for:

SOAPACTION_URI_PROPERTY

You will probably find code resembling:

requestContext.put(
    BindingProvider.SOAPACTION_URI_PROPERTY,
    soapActionUri
);

Now find where soapActionUri receives its value. It may come from this XML:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

That means the following happens:

${brkmidoffice.endpoint}
        ↓
https://wealth-uat.../BrkMidOfficeService
        ↓
Custom Spring 6 bean puts it into SOAPACTION_URI_PROPERTY
        ↓
HTTP request sends endpoint URL as SOAPAction
        ↓
Server expects GetMOTUserProfile
        ↓
SOAP Fault

What to send me next

Please send the Spring 6 custom bean code containing:

SOAPACTION_URI_PROPERTY

or the method named something like:

setSoapActionUri(...)

Also include approximately 10 lines above and below it.

We have already proven the mismatch. Step 2 will prove exactly which Spring 6 code is creating the incorrect header.