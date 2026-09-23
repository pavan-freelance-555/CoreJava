Worked for 27s

The confusion is around the word “null.”

Setting soapActionUri to null does not mean JAX-WS sends:

SOAPAction: null

It means:

> “We are not manually overriding SOAPAction. JAX-WS should choose the correct action using the WSDL operation.”



Two possible sources of SOAPAction

JAX-WS can obtain SOAPAction in two ways.

Method 1: Automatically from the WSDL

Your WSDL contains:

<wsdl:operation name="GetMOTUserProfile">
    <soap:operation soapAction="GetMOTUserProfile"/>
</wsdl:operation>

Your custom bean loads this WSDL:

Service service = Service.create(wsdlUrl, serviceQName);

proxy = service.getPort(
    portQName,
    serviceInterface
);

At this point, JAX-WS knows this mapping:

Java method:
getMOTUserProfile()

        ↓ maps to

WSDL operation:
GetMOTUserProfile

        ↓ has action

SOAPAction:
GetMOTUserProfile

When the application calls:

port.getMOTUserProfile(request);

JAX-WS can automatically generate:

SOAPAction: "GetMOTUserProfile"

This is the normal/default behavior.


---

Method 2: Manually force a SOAPAction

Your custom bean then executes:

bindingProvider.getRequestContext().put(
    BindingProvider.SOAPACTION_USE_PROPERTY,
    Boolean.TRUE
);

This tells JAX-WS:

> “I am manually supplying the SOAPAction.”



The next code provides that manual value:

bindingProvider.getRequestContext().put(
    BindingProvider.SOAPACTION_URI_PROPERTY,
    soapActionUri
);

But your XML puts the endpoint into soapActionUri:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

Therefore, JAX-WS’s automatic WSDL value is replaced:

Automatic value from WSDL:
GetMOTUserProfile

        ↓ overridden by custom code

Manual value:
https://wealth-uat.../BrkMidOfficeService

That causes the failure.

What happens when soapActionUri is null?

Your code contains:

if (soapActionUri != null && !soapActionUri.isBlank()) {
    // Manually override SOAPAction
}

If you remove this XML property:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

then:

soapActionUri == null

The condition becomes false:

if (false) {
    // This code is skipped
}

Therefore, these properties are never added:

SOAPACTION_USE_PROPERTY = true
SOAPACTION_URI_PROPERTY = endpointUrl

Nothing manually overrides the WSDL.

The flow becomes:

Application calls getMOTUserProfile()
        ↓
JAX-WS looks at the loaded WSDL
        ↓
Finds the GetMOTUserProfile operation
        ↓
Reads soapAction="GetMOTUserProfile"
        ↓
Sends SOAPAction: "GetMOTUserProfile"

So null means:

No manual override

It does not mean:

Send SOAPAction as null

Simple analogy

Imagine the WSDL is a GPS route.

The GPS automatically says:

Go to GetMOTUserProfile

But your custom code manually enters:

Go to https://.../BrkMidOfficeService

The manual instruction replaces the GPS instruction.

Removing the manual instruction does not leave the car without directions. It allows the GPS—the WSDL—to provide the correct direction again.

What happened in Spring 4?

Here I need to correct one earlier assumption:

> We have not yet proved that Spring 4 dropped or corrected the configured SOAPAction.



The Spring 4 XML also shows:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

Spring 4’s JaxWsPortProxyFactoryBean also had support for configuring soapActionUri. Therefore, we should not simply assume that Spring 4 ignored it.

There are several possibilities.

Possibility 1: Spring 4 sends GetMOTUserProfile

The old proxy/runtime may derive the operation action and not apply the configured value in the way your custom bean does.

Actual request:

SOAPAction: "GetMOTUserProfile"

Possibility 2: Spring 4 configuration resolves differently

The Spring 4 application might load another properties file:

Spring 4:
soapActionUri = GetMOTUserProfile

Spring 6:
soapActionUri = endpoint URL

Even though the XML placeholder names look the same, the resolved values or loaded configuration files may differ.

Possibility 3: Spring 4 also sends the wrong value

It might send:

SOAPAction: "https://.../BrkMidOfficeService"

but reach a server or route that accepts it.

Possibility 4: Another handler modifies the request

The old application could have a JAX-WS handler that changes the header after Spring prepares the proxy.

We cannot select one of these possibilities from the configuration screenshot alone.

How to prove Spring 4 behavior

Enable JAX-WS request dumping on Java 8:

-Dcom.sun.xml.internal.ws.transport.http.client.HttpTransportPipe.dump=true

Restart the Java 8 Tomcat and perform one login.

Search the log for:

SOAPAction

You should see something like:

SOAPAction: "GetMOTUserProfile"

or:

SOAPAction: "https://.../BrkMidOfficeService"

That one line tells us what Spring 4 actually sends.

Then enable the equivalent logging for Spring 6:

-Dcom.sun.xml.ws.transport.http.client.HttpTransportPipe.dump=true

You already expect to see:

SOAPAction: "https://.../BrkMidOfficeService"

Compare them:

Client	Actual SOAPAction

Spring 4	Capture from log
Spring 6	Endpoint URL
WSDL expects	GetMOTUserProfile


Recommended safe test

In the Spring 6 XML, temporarily remove only:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

Do not remove:

<property name="endpointAddress"
          value="${brkmidoffice.endpoint}"/>

Deploy first to DEV or UAT and enable request logging.

Expected output:

SOAPAction: "GetMOTUserProfile"

Then test another operation, such as UpdateFxRate.

Expected output:

SOAPAction: "UpdateFxRate"

That proves JAX-WS is selecting the action dynamically from the WSDL.

Final explanation

Current Spring 6 behavior:

WSDL selects GetMOTUserProfile
        ↓
Custom bean sets manual override=true
        ↓
Custom bean replaces it with endpoint URL
        ↓
Preprod rejects the request

After removing the manual property:

soapActionUri is null
        ↓
Custom override block is skipped
        ↓
JAX-WS uses the loaded WSDL metadata
        ↓
SOAPAction becomes GetMOTUserProfile

For Spring 4, we should capture the actual outgoing header before claiming whether it ignores, replaces or sends the configured value.