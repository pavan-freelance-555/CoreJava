Worked for 1m 10s

The simple explanation

These screenshots describe one migration problem that exposed several separate issues. They are not saying one bug caused everything.

The application moved from an older Java/Spring/JSF/Jersey setup to Java 21, Spring 6, JSF 4, Jersey 3, and Tomcat 10.1. During that move, the request first failed to reach Jersey. After routing was fixed, a separate Spring startup problem appeared. React errors on the external SecureShare page were a third issue.

First, follow what happens when someone uses SecureShare

1. The user selects SecureShare.


2. The application opens SecureShare.jsf. JSF builds the page.


3. The page calls the keepSessionAlive.rs endpoint to check the session.


4. It calls the token endpoint, shown in the screenshots as samISCORERefToken.rs.


5. The server creates or retrieves a SecureShare token.


6. The page submits that token to the external SecureShare site.


7. The external site opens.



Each step depends on the previous one. If the keep-alive request fails, the page may never reach the token step. If Jersey routes the request but Spring fails while creating a bean, the handler can be reached and the request can still fail afterward.

How the Jersey URL is assembled

The screenshots describe the public endpoint as:

/midofficetools-webapp/rs/brkoauth/keepSessionAlive.rs

Think of its parts like this:

URL part	What it means

/midofficetools-webapp	The web application’s context path
/rs	The Jersey servlet mapping in web.xml
/brkoauth	The resource class’s Jersey @Path
/keepSessionAlive.rs	The resource method’s Jersey @Path


Jersey servlet mappings provide the base path for the REST application; Jersey then matches the resource and method paths under it. 

The proposed setup is:

<!-- web.xml -->
<url-pattern>/rs/*</url-pattern>

// Resource class
@Path("/brkoauth")

// Resource method
@Path("/keepSessionAlive.rs")

Together, those still produce the same public URL:

/midofficetools-webapp/rs/brkoauth/keepSessionAlive.rs

Why change the class-level path in Java? Before the change, both web.xml and Java claimed the /rs part. If web.xml maps /rs/* and the Java class also says @Path("rs/brkoauth"), the rs portion is included twice in the route. The screenshot’s proposed fix moves ownership of /rs to web.xml; Java supplies only the remaining resource path.

Why not map Jersey to /*? That mapping catches far more than REST calls. It can send the home page, JSF pages, and static files to Jersey, which can make the home page stop working. /rs/* limits Jersey to requests beginning with /rs/. The screenshot reports that the older .rs mapping did not route this application’s request after the migration; it does not prove that every Jersey 3 application is unable to use an extension mapping.

What the 404 and 500 mean here

The 404 was a routing failure. The notes say the handler breakpoint was not reached. That means the request had not made it to the Java method yet. The routing fix is the explicit /rs/* Jersey entry point, with the class path adjusted to /brkoauth.

The later 500 was a different failure. Once Jersey could reach the endpoint, the token request started initializing Spring beans and failed. A 404 says “the route was not found”; a 500 says “the server encountered an error while processing a route.”

So the notes describe progress: first fix the route; then investigate the backend error that becomes visible.

Why FacesContext was null

FacesContext belongs to a JSF request. It is available while JSF is processing a JSF page. A Jersey REST call is processed by Jersey, not by the JSF servlet, so that call normally has no FacesContext.

The screenshots say the token flow created a second Spring context with code like:

new ClassPathXmlApplicationContext("beanRefContext.xml")

That context loaded a broad Spring XML configuration. Spring then eagerly created beans that were not needed for the token request, including UI or JSF controller beans. Some of those beans accessed FacesContext or servlet details during construction or initialization. During the Jersey request, FacesContext was null, so initialization failed and the endpoint returned a 500.

The same issue can happen if a constructor, field initializer, or @PostConstruct method reaches for JSF or servlet request information. Those operations can run before a JSF request exists—or while a Jersey request is being processed.

Short-term fix in the notes

Make unrelated UI beans lazy so Spring does not create them during token processing.

Move JSF-dependent work out of constructors and field initializers. Do it only when it is actually needed in a JSF page flow.

Pass required values, such as locale or configuration, into the code instead of obtaining them from FacesContext during initialization.


Lazy loading means Spring waits to create a bean until it is requested. It can stop an unrelated bean from breaking token startup, but it does not make FacesContext available during a Jersey call.

Stronger long-term direction

Use a backend service or a small Spring context for token work, and inject that service instead of creating a second full application context. Keep JSF page controllers separate from backend services. Backend services should not depend on FacesContext.

Why duplicate controller registration is risky

The notes describe PICBranchController as being defined in Spring XML and also discovered through @Named. That can create two Spring-managed instances of the same class.

The two instances can have different lifecycle or scope behavior. For example, JSF might use one instance while another receives Spring dependencies. The proposed minimal fix is to keep one registration. If the XML definition is the one that declares the intended session scope, remove the duplicate annotation-based registration.

What the React 418 and 423 errors mean

The screenshots treat these as separate errors on the external SecureShare React application.

In simple terms, React rendered some HTML on the server, then the browser started React and expected to see the same initial content. If the browser’s first render does not match the server HTML, React reports a hydration problem. It may then discard that initial HTML and render again in the browser.

The notes mention possible reasons such as different authentication or session state, browser-only values, date/time or locale differences, browser extensions, or invalid HTML. If both MidOfficeTools endpoints succeed and the token form is submitted, these React errors need investigation on the external page. They are not, by themselves, proof that Jersey or JSF is broken.

What to remember

The screenshots’ proposed Jersey fix changes both the servlet mapping and the Java class-level path:

<url-pattern>/rs/*</url-pattern>

@Path("/brkoauth")

That combination avoids sending the whole application to Jersey and preserves the public endpoint URL shown in the notes. A configuration-only change is possible only if you also accept a different route or add a server/proxy rewrite. To keep this URL and use the /rs/* mapping as shown, the class path must no longer repeat rs.

Interview-ready explanation

> During the Java 21, Spring 6, JSF 4, and Jersey 3 migration, the legacy .rs servlet mapping stopped routing the REST request to its Jersey handler, so the breakpoint was not reached and the request returned 404. We added a dedicated /rs/* Jersey mapping to avoid catching the JSF home page, and changed the Jersey class path to /brkoauth so /rs was defined only once and the external URL stayed unchanged. After routing worked, a separate 500 was exposed: the token request created a second, broad Spring context that eagerly initialized JSF controllers. Those controllers accessed FacesContext, which does not exist during a Jersey request. We made unrelated UI beans lazy, removed duplicate controller registration, and identified a backend-only Spring context as the stronger long-term approach. React hydration errors on the external SecureShare page were tracked separately.