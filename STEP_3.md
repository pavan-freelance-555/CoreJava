Worked for 18s

Now we have concrete proof from your Spring 6 custom bean.

What the Spring 6 code does

First, it correctly sets the destination:

bindingProvider.getRequestContext().put(
    BindingProvider.ENDPOINT_ADDRESS_PROPERTY,
    endpointAddress
);

Then lines 79–88 explicitly force the SOAPAction:

if (soapActionUri != null && !soapActionUri.isBlank()) {
    bindingProvider.getRequestContext().put(
        BindingProvider.SOAPACTION_USE_PROPERTY,
        Boolean.TRUE
    );

    bindingProvider.getRequestContext().put(
        BindingProvider.SOAPACTION_URI_PROPERTY,
        soapActionUri
    );
}

Your XML passes:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

Therefore, the Spring 6 flow is:

${brkmidoffice.endpoint}
        ↓
https://wealth-uat.../BrkMid

Now we have found the exact Spring 6 code causing the problem.

Step 2 result: root cause confirmed

Your custom bean executes:

if (soapActionUri != null && !soapActionUri.isBlank()) {

    bindingProvider.getRequestContext().put(
        BindingProvider.SOAPACTION_USE_PROPERTY,
        Boolean.TRUE
    );

    bindingProvider.getRequestContext().put(
        BindingProvider.SOAPACTION_URI_PROPERTY,
        soapActionUri
    );
}

Your XML supplies:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

Therefore, the complete flow is:

brkmidoffice.endpoint
        ↓
https://wealth-uat.../BrkMidOfficeService
        ↓
setSoapActionUri(endpoint URL)
        ↓
SOAPACTION_USE_PROPERTY = true
        ↓
Spring 6 forcibly sends the endpoint URL as SOAPAction
        ↓
Preprod expects "GetMOTUserProfile"
        ↓
SOAP Fault

The evidence now matches exactly:

Evidence	Value

WSDL expects	GetMOTUserProfile
Custom bean sends	${brkmidoffice.endpoint}
Resolved value	https://wealth-uat.../BrkMidOfficeService
Server response	SOAPAction does not match an operation


Correct fix

Because this WSDL contains multiple operations, such as:

GetMOTUserProfile
UpdateFxRate
UpdateClientAttributes

you should not configure one global SOAPAction for the complete service.

Remove this XML property:

<property name="soapActionUri"
          value="${brkmidoffice.endpoint}"/>

Then soapActionUri will be null, so this block will not execute:

if (soapActionUri != null && !soapActionUri.isBlank()) {
    // Will be skipped
}

The JAX-WS proxy can then use the WSDL action corresponding to each called method:

getMOTUserProfile() → GetMOTUserProfile
updateFxRate()      → UpdateFxRate

Do not replace the XML value globally with:

value="GetMOTUserProfile"

because other operations would then incorrectly send the same action.

Step 3: Prove what Spring 4 sends

We still need to prove why Spring 4 works even though its XML has the same suspicious property.

Temporarily add this option to the Java 8 Tomcat JVM:

-Dcom.sun.xml.internal.ws.transport.http.client.HttpTransportPipe.dump=true

Restart only the Java 8 Tomcat and perform one login.

Search the Tomcat log for:

SOAPAction

We need to see whether Java 8 actually sends:

SOAPAction: "GetMOTUserProfile"

or:

SOAPAction: "https://.../BrkMidOfficeService"

How to interpret it

If Java 8 sends:

SOAPAction: "GetMOTUserProfile"

then the old Spring/JAX-WS implementation ignored or replaced the configured URL using the WSDL.

If Java 8 sends the endpoint URL and still works, then the old request is reaching a different or more tolerant backend.

For now, the Spring 6 failure itself is confirmed: lines 79–88 force the endpoint URL into the SOAPAction header, overriding the correct GetMOTUserProfile action from the WSDL.