Worked for 23s

The SecureShare command link is probably working correctly.

It is navigating to SecureShare.xhtml, but the JavaScript inside SecureShare.xhtml calls the token REST API. When that call fails, your own error function explicitly sends the browser to the login page:

window.location.href =
    "/midofficetools-webapp/login/Login.jsf";

So the actual flow is:

Click SecureShare
        ↓
handleClick()
        ↓
displayView()
        ↓
SecureShare.xhtml loads
        ↓
body onload calls cmdPost()
        ↓
Token Ajax request fails
        ↓
error callback executes
        ↓
JavaScript redirects to Login.jsf

This is not primarily an actionListener, navigation, or Faces 4 problem. The failure is inside the JavaScript/SSO-token flow.

Problems visible in the current SecureShare.xhtml

1. Hard-coded context path

Your code appears to use:

url: "/midofficetools-webapp/rs/brkauth/..."

and:

window.location.href =
    "/midofficetools-webapp/login/Login.jsf";

If the Faces 4 application is deployed under a different context root, these URLs are wrong.

Use the actual context path:

#{facesContext.externalContext.requestContextPath}

2. Request parameters are in the wrong place

Your current jQuery code appears similar to:

$.ajax({
    type: "GET",
    url: "...",
    dataType: "json",
    channel: "SCORE",
    forceToRefresh: true
});

These two values:

channel: "SCORE",
forceToRefresh: true

are not request parameters when placed directly inside the jQuery configuration object.

They should be inside data:

data: {
    channel: "SCORE",
    forceToRefresh: true
}

The old endpoint may require these parameters. Without them, the token call may fail.

3. Every error is treated as session expiration

Currently, any error causes:

window.location.href = ".../login/Login.jsf";

But the error might be:

Wrong URL

HTTP 404

HTTP 500

JSON parsing failure

Missing parameters

CORS failure

Token service unavailable

Session expired

Response is HTML instead of JSON


Only authentication errors such as 401 or 403 should immediately redirect to login.

4. Old jQuery is manually loaded

Your page loads something similar to:

<script src="../resources/js/jquery/jquery-1.12.4.js">
</script>

PrimeFaces 15 uses a much newer compatible jQuery version. Loading an old version manually can create conflicts.

For this standalone page, the cleanest solution is to use the browser’s native fetch() API. Then this page does not depend on jQuery at all.

Recommended Faces 4 SecureShare.xhtml

Use this as the updated structure. Keep the exact REST endpoint spellings from your application if they differ slightly from the screenshot.

<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="jakarta.faces.html"
      xmlns:f="jakarta.faces.core"
      xmlns:ui="jakarta.faces.facelets">

<h:head>

    <title>SecureShare</title>

</h:head>

<h:body>

    <h:outputText
            id="loadingMessage"
            value="Loading SecureShare..."/>

    <!--
        Plain HTML form is intentional.

        This form submits to an external SecureShare/Cloud endpoint,
        so it should not be changed to h:form.
    -->
    <form id="cdmForm"
          action="#{
              clientAccountNavBean.secureShareCloudBaseAPI
          }"
          method="post">

        <input id="channel"
               name="channel"
               type="hidden"
               value="SCORE"/>

        <input id="token"
               name="SessionToken"
               type="hidden"
               value=""/>

        <input id="buzLine"
               name="buzLine"
               type="hidden"
               value="ITRADE"/>

    </form>

    <script type="text/javascript">
        //<![CDATA[

        document.addEventListener(
            "DOMContentLoaded",
            function () {
                openSecureShare();
            }
        );

        async function openSecureShare() {

            const contextPath =
                "#{facesContext.externalContext.requestContextPath}";

            const tokenUrl =
                contextPath
                + "/rs/brkauth/samlSCORERefToken.rs"
                + "?channel="
                + encodeURIComponent("SCORE")
                + "&forceToRefresh=true";

            const loginUrl =
                contextPath + "/login/Login.jsf";

            try {

                console.log(
                    "Requesting SecureShare token from:",
                    tokenUrl
                );

                const response = await fetch(
                    tokenUrl,
                    {
                        method: "GET",

                        /*
                         * Send the current application session cookie.
                         */
                        credentials: "same-origin",

                        cache: "no-store",

                        headers: {
                            "Accept": "application/json"
                        }
                    }
                );

                console.log(
                    "SecureShare token HTTP status:",
                    response.status
                );

                /*
                 * Redirect to login only for actual
                 * authentication/authorization failures.
                 */
                if (response.status === 401
                        || response.status === 403) {

                    window.location.replace(loginUrl);
                    return;
                }

                if (!response.ok) {
                    throw new Error(
                        "Token service returned HTTP "
                        + response.status
                    );
                }

                const contentType =
                    response.headers.get("content-type")
                    || "";

                /*
                 * A session/security filter may return a login page
                 * as HTML instead of returning JSON.
                 */
                if (!contentType.includes(
                        "application/json")) {

                    const unexpectedBody =
                        await response.text();

                    console.error(
                        "Expected JSON but received:",
                        contentType,
                        unexpectedBody.substring(0, 300)
                    );

                    throw new Error(
                        "Token service returned a non-JSON response"
                    );
                }

                const tokenResponse =
                    await response.json();

                console.log(
                    "SecureShare token response received"
                );

                if (!tokenResponse.access_token) {
                    throw new Error(
                        "Token response does not contain access_token"
                    );
                }

                document.getElementById("token").value =
                    tokenResponse.access_token;

                /*
                 * Submit the token to the external SecureShare
                 * endpoint configured by the backing bean.
                 */
                document.getElementById("cdmForm").submit();

            } catch (error) {

                console.error(
                    "Unable to open SecureShare:",
                    error
                );

                const loadingMessage =
                    document.getElementById(
                        "loadingMessage"
                    );

                if (loadingMessage) {
                    loadingMessage.textContent =
                        "Unable to open SecureShare. "
                        + "Please try again or contact support.";
                }

                /*
                 * Do not redirect to login for every error.
                 * This allows you to see the real failure.
                 */
            }
        }

        //]]>
    </script>

</h:body>

</html>

Verify the exact parameter names

Your REST endpoint may expect:

channel
forceToRefresh

The new URL sends:

...?channel=SCORE&forceToRefresh=true

If your REST method expects different names, use those exact names.

For example, if the endpoint is:

@GET
public Response getToken(
    @QueryParam("channel") String channel,
    @QueryParam("forceToRefresh") boolean refresh) {
}

then the proposed URL is correct.

If you want to keep jQuery

If you do not want to replace the JavaScript with fetch(), use this corrected Ajax request:

<script type="text/javascript">
    //<![CDATA[

    function cmdPost() {

        const contextPath =
            "#{facesContext.externalContext.requestContextPath}";

        $.ajax({
            type: "GET",

            url: contextPath
                + "/rs/brkauth/samlSCORERefToken.rs",

            dataType: "json",

            data: {
                channel: "SCORE",
                forceToRefresh: true
            },

            success: function (response) {

                if (!response.access_token) {
                    console.error(
                        "access_token is missing",
                        response
                    );
                    return;
                }

                document.getElementById("token").value =
                    response.access_token;

                document.getElementById("cdmForm")
                        .submit();
            },

            error: function (
                    xhr,
                    textStatus,
                    errorThrown) {

                console.error(
                    "SecureShare token request failed",
                    {
                        status: xhr.status,
                        textStatus: textStatus,
                        error: errorThrown,
                        responseText: xhr.responseText
                    }
                );

                if (xhr.status === 401
                        || xhr.status === 403) {

                    window.location.replace(
                        contextPath
                        + "/login/Login.jsf"
                    );
                    return;
                }

                document.getElementById(
                    "loadingMessage"
                ).textContent =
                    "Unable to open SecureShare. "
                    + "HTTP status: "
                    + xhr.status;
            }
        });
    }

    //]]>
</script>

The critical correction is:

data: {
    channel: "SCORE",
    forceToRefresh: true
}

instead of:

channel: "SCORE",
forceToRefresh: true

Check the browser Network tab

After making the diagnostic change, click SecureShare and open browser Developer Tools → Network.

You should see this sequence:

Client Account POST
        ↓
SecureShare.xhtml or SecureShare.jsf
        ↓
samlSCORERefToken.rs
        ↓
External SecureShare endpoint

Inspect the token request.

If the status is 404

The endpoint URL is wrong.

Confirm:

Context path
REST base path
Endpoint spelling
Uppercase/lowercase characters
.rs mapping

The request should use:

actual-context-root
    + /rs/brkauth/samlSCORERefToken.rs

If the status is 401 or 403

The request is not reaching the REST endpoint with a valid authenticated session.

Check:

Does the request include the session cookie?

Is the REST URL under the same hostname and context?

Does the security filter permit the endpoint?

Has the session expired?

Is the new window using the same application hostname?


With:

credentials: "same-origin"

the same-origin session cookie should be included.

If the status is 302

Inspect the response headers:

Location: /login/Login.jsf

That means a filter or SSO layer is redirecting the token REST call to login before the endpoint executes.

The fix is then in the authentication/filter configuration, not in Faces rendering.

If status is 200 but response is HTML

The security layer may have followed a redirect and returned the login page:

<html>
    Login page...
</html>

But your JavaScript expects JSON:

{
  "access_token": "..."
}

This causes JSON parsing failure and enters the error handler.

If status is 200 JSON without access_token

The token endpoint succeeded but returned a different structure.

For example:

{
  "accessToken": "..."
}

Then update:

tokenResponse.access_token

to the actual property:

tokenResponse.accessToken

Keepalive request

Your old page also appears to call:

keepSessionAlive.rs

Do not mix that change with the first diagnosis.

First verify this flow:

SecureShare page
    ↓
Token API returns JSON
    ↓
access_token populated
    ↓
cdmForm submitted

Once token navigation works, add the keepalive call back if it is still required.

A keepalive call should not redirect to login for every possible technical failure. Log non-authentication failures separately.

One more navigation check

Your displayView() action must navigate to the Faces URL, for example:

/SecureShare.jsf

or an outcome mapped to it.

Faces 4 may use an .xhtml or extensionless mapping depending on your FacesServlet configuration.

Check web.xml:

<servlet-mapping>
    <servlet-name>Faces Servlet</servlet-name>
    <url-pattern>*.jsf</url-pattern>
</servlet-mapping>

If the mapping is:

<url-pattern>*.jsf</url-pattern>

then the browser-facing URL is:

SecureShare.jsf

while the physical file remains:

SecureShare.xhtml

Do not rename the physical file to .jsf.

Most likely root cause

Based on the attached code, the most likely sequence is:

SecureShare.xhtml successfully loads
        ↓
cmdPost() executes
        ↓
samlSCORERefToken REST call fails
        ↓
Existing error callback executes
        ↓
window.location.href points to Login.jsf

Start by making these three changes:

1. Build the REST and login URLs using the runtime context path.


2. Send channel and forceToRefresh as actual request parameters.


3. Redirect to login only for 401/403; log and display all other errors.



That will either fix the problem directly or reveal the exact failing HTTP status instead of hiding every failure behind the login page.