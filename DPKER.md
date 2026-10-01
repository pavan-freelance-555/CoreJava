No. When converting LocalDate to XMLGregorianCalendar, you do not need to specify a text format such as MM/dd/yyyy.

A format is required only when converting between a date object and a String.

Recommended conversion

import java.time.LocalDate;
import javax.xml.datatype.DatatypeConstants;
import javax.xml.datatype.DatatypeFactory;
import javax.xml.datatype.XMLGregorianCalendar;

public static XMLGregorianCalendar toXmlGregorianCalendar(
        LocalDate localDate) {

    if (localDate == null) {
        return null;
    }

    try {
        return DatatypeFactory.newInstance()
                .newXMLGregorianCalendarDate(
                        localDate.getYear(),
                        localDate.getMonthValue(),
                        localDate.getDayOfMonth(),
                        DatatypeConstants.FIELD_UNDEFINED
                );
    } catch (Exception exception) {
        throw new IllegalStateException(
                "Unable to convert LocalDate to XMLGregorianCalendar",
                exception
        );
    }
}

This passes numeric values directly:

Year  = 2026
Month = 9
Day   = 30

Therefore, no date pattern is involved.

The resulting XML value is normally:

<effectiveDate>2026-09-30</effectiveDate>

That format—yyyy-MM-dd—is the XML Schema standard representation for xs:date. You should not force it to MM/dd/yyyy.

Where the UI pattern applies

This pattern:

<p:datePicker
    value="#{transferPlan.planBean.effectiveDate}"
    pattern="MM/dd/yyyy" />

controls only how the user sees and enters the date:

09/30/2026

Internally it is still:

LocalDate.of(2026, 9, 30)

When sent to the SOAP service it becomes:

2026-09-30

These are the same date, displayed in different representations.

When a formatter is needed

A formatter is needed only if you first receive a String:

String dateText = "09/30/2026";

Then you must parse it:

DateTimeFormatter formatter =
        DateTimeFormatter.ofPattern("MM/dd/yyyy");

LocalDate localDate =
        LocalDate.parse(dateText, formatter);

After it becomes LocalDate, convert it to XMLGregorianCalendar without a formatter.

If the service expects date and time

If the generated service field represents xs:dateTime, it may expect a value such as:

2026-09-30T00:00:00

Use:

public static XMLGregorianCalendar toXmlGregorianDateTime(
        LocalDate localDate) {

    if (localDate == null) {
        return null;
    }

    try {
        return DatatypeFactory.newInstance()
                .newXMLGregorianCalendar(
                        localDate.getYear(),
                        localDate.getMonthValue(),
                        localDate.getDayOfMonth(),
                        0,
                        0,
                        0,
                        0,
                        DatatypeConstants.FIELD_UNDEFINED
                );
    } catch (Exception exception) {
        throw new IllegalStateException(
                "Unable to convert date",
                exception
        );
    }
}

Choose based on the generated SOAP request setter or WSDL:

WSDL xs:date → use newXMLGregorianCalendarDate(...)

WSDL xs:dateTime → use newXMLGregorianCalendar(...)


So the key point is: the WSDL type determines whether you include time; a display format does not control the object conversion.