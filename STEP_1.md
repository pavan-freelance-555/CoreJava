Worked for 11s

Let us do this one step at a time. Do not change any code yet.

Step 1: Find what SOAPAction the WSDL expects

The server error says:

The given SOAPAction does not match an operation

So first, we must find the correct SOAPAction for:

getMOTUserProfile

1. Locate the WSDL

Your Spring configuration shows this location:

classpath:com/bns/xsd/msgs/brkmidoffice/v001/BrkMidOffice.wsdl

In your source code, it will probably be under:

src/main/resources/com/bns/xsd/msgs/brkmidoffice/v001/BrkMidOffice.wsdl

2. Open BrkMidOffice.wsdl

Open the file in Eclipse, IntelliJ or Notepad++.

Search for:

getMOTUserProfile

You may find multiple occurrences. Continue searching until you see something similar to:

<wsdl:operation name="getMOTUserProfile">
    <soap:operation soapAction="something"/>

For example:

<wsdl:operation name="getMOTUserProfile">
    <soap:operation
        soapAction="http://xsd.bns.com/msgs/brkmidoffice/v001/getMOTUserProfile"/>
</wsdl:operation>

Or it could contain:

<soap:operation soapAction=""/>

3. Record the exact value

Write down exactly what is inside:

soapAction="..."

Do not change capitalization, slash, http, https, or spaces.

We need this comparison:

Expected by WSDL:
______________________________

Currently received by server:
https://wealth-uat.../BrkMidOfficeService

4. If you are checking from Linux

Run:

grep -n -A5 -B5 \
'operation name="getMOTUserProfile"' \
BrkMidOffice.wsdl

If you do not know where the file is:

find /your/application/path \
-name "BrkMidOffice.wsdl"

Then use the returned path:

grep -n -A5 -B5 \
'operation name="getMOTUserProfile"' \
/full/path/BrkMidOffice.wsdl

Result expected from Step 1

At the end of this step, we need only one piece of information:

WSDL SOAPAction for getMOTUserProfile = ?

Please send me a screenshot of the WSDL section containing:

<wsdl:operation name="getMOTUserProfile">

and:

<soap:operation soapAction="...">

Then I will give you only Step 2, which will show how to capture the SOAPAction actually sent by Spring 4 and Spring 6.