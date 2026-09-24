<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="jakarta.faces.html"
      xmlns:f="jakarta.faces.core">

<h:head>

    <title>SecureShare</title>

    <meta charset="UTF-8"/>

</h:head>

<h:body>

    <div id="statusMessage">
        Loading SecureShare...
    </div>

    <!--
        This is intentionally a plain HTML form.

        It posts the generated SSO token to the external
        SecureShare application. Do not change it to h:form.
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

        /*
         * Use the deployed application context dynamically.
         *
         * For example:
         *
         * /midofficetools-webapp
         */
        const applicationContext =
            "#{facesContext.externalContext.requestContextPath}";

        const loginUrl =
            applicationContext + "/login/Login.jsf";

        const tokenUrl =
            applicationContext
            + "/rs/brkauth/samlSCORERefToken.rs";

        const keepAliveUrl =
            applicationContext
            + "/rs/brkauth/keepSessionAlive.rs";

        /*
         * Start after the complete HTML document is available.
         */
        document.addEventListener(
            "DOMContentLoaded",
            function () {
                openSecureShare();
            }
        );

        async function openSecureShare() {

            setStatusMessage(
                "Preparing SecureShare session..."
            );

            /*
             * Keep the application session alive.
             *
             * This is treated as a best-effort call so that
             * a technical problem in keepalive does not hide
             * the real token API result.
             */
            sendKeepAlive();

            try {

                const requestUrl =
                    new URL(
                        tokenUrl,
                        window.location.origin
                    );

                /*
                 * These were incorrectly placed directly inside
                 * $.ajax in the old code.
                 *
                 * They must be actual query parameters.
                 */
                requestUrl.searchParams.set(
                    "channel",
                    "SCORE"
                );

                requestUrl.searchParams.set(
                    "forceToRefresh",
                    "true"
                );

                console.log(
                    "Calling SecureShare token endpoint:",
                    requestUrl.toString()
                );

                const response = await fetch(
                    requestUrl.toString(),
                    {
                        method: "GET",

                        /*
                         * Include the existing application
                         * session cookie.
                         */
                        credentials: "same-origin",

                        cache: "no-store",

                        headers: {
                            "Accept": "application/json"
                        }
                    }
                );

                console.log(
                    "Token API status:",
                    response.status
                );

                /*
                 * Redirect only when the server confirms that
                 * authentication/authorization failed.
                 */
                if (response.status === 401
                        || response.status === 403) {

                    redirectToLogin();
                    return;
                }

                if (!response.ok) {
                    throw new Error(
                        "Token API returned HTTP "
                        + response.status
                    );
                }

                const contentType =
                    response.headers.get(
                        "content-type"
                    ) || "";

                /*
                 * Sometimes an authentication filter returns
                 * the login HTML page with HTTP 200.
                 *
                 * Do not try to parse HTML as JSON.
                 */
                if (!contentType
                        .toLowerCase()
                        .includes("application/json")) {

                    const responseBody =
                        await response.text();

                    console.error(
                        "Expected JSON but received "
                        + contentType,
                        responseBody.substring(0, 500)
                    );

                    if (looksLikeLoginPage(responseBody)) {
                        redirectToLogin();
                        return;
                    }

                    throw new Error(
                        "Token API returned a non-JSON response"
                    );
                }

                const tokenResponse =
                    await response.json();

                /*
                 * Do not print the actual token in logs.
                 */
                console.log(
                    "SecureShare token response received"
                );

                if (!tokenResponse
                        || !tokenResponse.access_token) {

                    throw new Error(
                        "Token API response does not contain "
                        + "access_token"
                    );
                }

                const tokenInput =
                    document.getElementById("token");

                const cdmForm =
                    document.getElementById("cdmForm");

                if (!tokenInput) {
                    throw new Error(
                        "SessionToken hidden input was not found"
                    );
                }

                if (!cdmForm) {
                    throw new Error(
                        "cdmForm was not found"
                    );
                }

                tokenInput.value =
                    tokenResponse.access_token;

                setStatusMessage(
                    "Opening SecureShare..."
                );

                /*
                 * Submit the token to the external
                 * SecureShare URL.
                 */
                cdmForm.submit();

            } catch (error) {

                console.error(
                    "SecureShare initialization failed:",
                    error
                );

                setStatusMessage(
                    "Unable to open SecureShare. "
                    + "Please try again or contact support."
                );
            }
        }

        async function sendKeepAlive() {

            try {

                const response = await fetch(
                    keepAliveUrl,
                    {
                        method: "POST",

                        credentials: "same-origin",

                        cache: "no-store",

                        keepalive: true,

                        headers: {
                            "Accept": "application/json",
                            "Content-Type":
                                "application/json"
                        },

                        body: JSON.stringify({
                            channel: "SCORE"
                        })
                    }
                );

                console.log(
                    "Keepalive API status:",
                    response.status
                );

                if (response.status === 401
                        || response.status === 403) {

                    console.warn(
                        "Keepalive detected an expired session"
                    );

                    redirectToLogin();
                    return;
                }

                if (!response.ok) {
                    console.warn(
                        "Keepalive returned HTTP "
                        + response.status
                    );
                }

            } catch (error) {

                /*
                 * Do not immediately redirect to login for a
                 * network or keepalive technical failure.
                 */
                console.warn(
                    "Keepalive request failed:",
                    error
                );
            }
        }

        function redirectToLogin() {

            console.warn(
                "Redirecting to login because the "
                + "session is unauthorized or expired"
            );

            window.location.replace(loginUrl);
        }

        function looksLikeLoginPage(responseBody) {

            if (!responseBody) {
                return false;
            }

            const normalizedBody =
                responseBody.toLowerCase();

            return normalizedBody.includes(
                        "<html"
                    )
                    && (
                        normalizedBody.includes(
                            "login"
                        )
                        || normalizedBody.includes(
                            "signin"
                        )
                        || normalizedBody.includes(
                            "sign in"
                        )
                    );
        }

        function setStatusMessage(message) {

            const statusElement =
                document.getElementById(
                    "statusMessage"
                );

            if (statusElement) {
                statusElement.textContent = message;
            }
        }

        //]]>
    </script>

</h:body>

</html>