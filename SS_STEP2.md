Worked for 12s

The key change is not “remove every f:event.” The correct rule is:

> Remove f:event type="preRenderComponent" only where you are using it to call component.setRendered(false).



f:event is still supported in Jakarta Faces 4. The problem is the timing of that specific event for your requirement.

For your application, the cleaner design is:

One f:viewAction at page level
        ↓
Load all component permissions once
        ↓
Each component uses rendered="#{...}"
        ↓
Keep actionListener and action on command links
        ↓
Check authorization again in the action

Also, the replacement is called:

<f:viewAction>

not f:event action.

1. Current JSF 2 design

Your current SecureShare code is approximately:

<h:panelGroup
        layout="block"
        id="secureShareLinkDiv">

    <li>
        <ui:insert name="secureShareLinkDef">

            <h:commandLink
                    id="secureShareLink"
                    value="#{tBundle.SECURE_SHARE}"
                    disabled="#{
                        clientAccountNavBean
                            .disabledStatusMap['secureShareLink']
                    }"
                    styleClass="#{
                        clientAccountNavBean
                            .styleClassMap['secureShareLink']
                    }"
                    actionListener="#{
                        clientAccountNavBean.handleClick
                    }"
                    target="SecureShareCDM"
                    action="#{
                        clientAccountNavBean.displayView
                    }"/>

        </ui:insert>
    </li>

    <f:event
            type="preRenderComponent"
            listener="#{
                applicationAccessBean.checkComponentAccess
            }"/>

</h:panelGroup>

The intention is:

Before rendering secureShareLinkDiv
        ↓
Call checkComponentAccess()
        ↓
Check entitlement
        ↓
If unauthorized:
component.setRendered(false)

The intention is correct, but the event is too closely tied to the component’s rendering process.

2. What preRenderComponent actually means

preRenderComponent does not mean:

> Check permission before JSF starts processing this component.



It means:

> JSF has reached this component during Render Response and is about to encode it.



The approximate flow is:

Restore/build JSF view
        ↓
Create secureShareLinkDiv
        ↓
Create secureShareLink
        ↓
Evaluate some component attributes
        ↓
Enter Render Response
        ↓
JSF checks whether component is rendered
        ↓
JSF begins component rendering
        ↓
PreRenderComponentEvent fires
        ↓
checkComponentAccess() executes
        ↓
Listener calls setRendered(false)

The critical point is:

JSF may already have checked rendered=true
before the preRenderComponent event runs

In some old Mojarra 2 versions, changing rendered at this point appeared to work because the implementation checked the property again during encoding.

Faces 4 may not repeat the same check at exactly the same point. The application was depending on an implementation detail rather than a strong lifecycle guarantee.

That explains why you can see:

component.setRendered(false);

but the component still appears.

3. Why it worked in JSF 2

The old flow may have been:

JSF checks rendered=true
        ↓
preRenderComponent event
        ↓
setRendered(false)
        ↓
Old Mojarra checks rendered again
        ↓
Component skipped

The Faces 4 flow may effectively behave like:

JSF checks rendered=true
        ↓
Decides to render component
        ↓
preRenderComponent event
        ↓
setRendered(false)
        ↓
Encoding has already started

Faces 4 did not redefine preRenderComponent as an after-render event. It still occurs before final component encoding.

But “before final encoding” is not the same as:

before JSF decides whether the component participates

That is why it is better to provide the correct rendered value before Render Response begins.

4. Why f:viewAction solves it

f:viewAction is a page-level action.

We use it to prepare the permission information earlier:

<f:metadata>
    <f:viewAction
            action="#{applicationAccessBean.prepareAccess}"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>
</f:metadata>

Now the lifecycle becomes:

Restore/build basic JSF view
        ↓
Apply Request Values
        ↓
prepareAccess() executes
        ↓
Permission map is prepared
        ↓
Process Validations
        ↓
Update Model Values
        ↓
Invoke Application
        ↓
Render Response
        ↓
JSF evaluates rendered expression
        ↓
Allowed?
  ├── No  → Skip component completely
  └── Yes → Encode component

Now JSF knows the correct value before it decides to render the component.

5. f:viewAction does not render anything

It is important to understand the different responsibilities.

f:viewAction

Used to prepare the page:

prepareAccess();

It can:

Load permissions

Load initial page data

Validate a request

Redirect

Select the correct page mode


rendered

Used to decide component visibility:

rendered="#{
    applicationAccessBean.canRender('secureShareLink')
}"

actionListener

Used after the user clicks:

handleClick(ActionEvent event);

It can:

Identify the clicked link

Change selected-menu styling

Set navigation state

Log the click


action

Used after the listener:

displayView();

It can:

Recheck authorization

Execute business logic

Return navigation outcome


These responsibilities should remain separate.

6. Where to put f:metadata

Place it directly below the root f:view, before html.

<!DOCTYPE html>

<f:view xmlns="http://www.w3.org/1999/xhtml"
        xmlns:f="jakarta.faces.core"
        xmlns:h="jakarta.faces.html"
        xmlns:ui="jakarta.faces.facelets"
        xmlns:p="primefaces"
        xmlns:mot="http://www.midofficetools.bns.com/facelet/components">

    <f:metadata>

        <f:viewAction
                action="#{
                    applicationAccessBean.prepareAccess
                }"
                phase="APPLY_REQUEST_VALUES"
                onPostback="false"/>

        <ui:insert name="metadata"/>

    </f:metadata>

    <html lang="en">
        ...
    </html>

</f:view>

Do not put it inside:

<h:head>

Do not put it inside:

<h:body>

Do not put it inside:

<h:form>

Do not place it beside each one of the 65 components.

You need:

One f:metadata
    ↓
One f:viewAction
    ↓
Permission data for all secured components

7. Should f:view be removed?

No.

Keep the root:

<f:view>

Only update its namespaces.

JSF 2:

xmlns:f="http://java.sun.com/jsf/core"
xmlns:h="http://java.sun.com/jsf/html"
xmlns:ui="http://java.sun.com/jsf/facelets"

Faces 4:

xmlns:f="jakarta.faces.core"
xmlns:h="jakarta.faces.html"
xmlns:ui="jakarta.faces.facelets"

PrimeFaces:

xmlns:p="primefaces"

Your custom namespace can remain:

xmlns:mot=
    "http://www.midofficetools.bns.com/facelet/components"

The Java classes behind mot: must migrate from javax.faces.* to jakarta.faces.*.

8. Complete base template

A migrated BasePageTemplate.xhtml can look like this:

<!DOCTYPE html>

<f:view xmlns="http://www.w3.org/1999/xhtml"
        xmlns:f="jakarta.faces.core"
        xmlns:h="jakarta.faces.html"
        xmlns:ui="jakarta.faces.facelets"
        xmlns:p="primefaces"
        xmlns:mot="http://www.midofficetools.bns.com/facelet/components"
        contentType="text/html"
        encoding="UTF-8">

    <f:metadata>

        <!-- Executes once for the initial page request -->
        <f:viewAction
                action="#{
                    applicationAccessBean.prepareAccess
                }"
                phase="APPLY_REQUEST_VALUES"
                onPostback="false"/>

        <!-- Allows child pages to provide additional metadata -->
        <ui:insert name="metadata"/>

    </f:metadata>

    <html lang="en">

    <h:head>

        <meta charset="UTF-8"/>

        <meta name="viewport"
              content="width=device-width, initial-scale=1.0"/>

        <meta http-equiv="Cache-Control"
              content="no-cache, no-store, must-revalidate"/>

        <meta http-equiv="Pragma"
              content="no-cache"/>

        <meta http-equiv="Expires"
              content="0"/>

        <title>
            <ui:insert name="pageTitle">
                #{tBundle.WINDOW_TITLE}
            </ui:insert>
        </title>

        <ui:insert name="html-head-extra-meta"/>

        <h:outputScript
                library="js"
                name="midofficetools.js"
                target="head"/>

        <!-- Do not load jQuery 1.8.2 with PrimeFaces 15 -->

        <h:outputStylesheet
                library="css"
                name="midofficetools.css"/>

        <h:outputStylesheet
                library="css"
                name="word.css"/>

        <ui:insert name="additional-head-content"/>

    </h:head>

    <h:body>

        <f:loadBundle
                basename="application"
                var="tBundle"/>

        <f:loadBundle
                basename="application_gif"
                var="gBundle"/>

        <h:panelGroup
                id="maincontainer"
                layout="block">

            <h:panelGroup
                    id="header"
                    layout="block">

                <ui:insert name="header">
                    <ui:include src="/template/Header.xhtml"/>
                </ui:insert>

            </h:panelGroup>

            <h:panelGroup
                    id="topNavigation"
                    layout="block"
                    styleClass="nav-top-lvl">

                <ui:insert name="nav-top-lvl">
                    <ui:include
                            src="/template/MainNavigationBar.xhtml"/>
                </ui:insert>

            </h:panelGroup>

            <h:panelGroup
                    id="contentwrapper"
                    layout="block">

                <h:panelGroup
                        id="contentcolumn"
                        layout="block">

                    <ui:insert name="main-content">
                        <ui:include
                                src="/template/MainContent.xhtml"/>
                    </ui:insert>

                </h:panelGroup>

                <h:panelGroup
                        id="leftcolumn"
                        layout="block"
                        styleClass="nav-sub-lvl">

                    <!--
                        The navigation has its own form so that
                        unrelated input validation does not prevent
                        a menu command from executing.
                    -->
                    <h:form
                            id="leftNavigationForm"
                            prependId="false">

                        <ui:insert name="left-nav">
                            <ui:include
                                    src="/template/LeftNavigationBar.xhtml"/>
                        </ui:insert>

                    </h:form>

                </h:panelGroup>

            </h:panelGroup>

            <h:panelGroup
                    id="footer"
                    layout="block"
                    styleClass="clearboth space1">

                <ui:insert name="footer">
                    <ui:include src="/template/Footer.xhtml"/>
                </ui:insert>

            </h:panelGroup>

        </h:panelGroup>

    </h:body>

    </html>

</f:view>

9. Complete SecureShare section

Keep the standard Faces h:commandLink. PrimeFaces 15 does not require converting it.

<ui:composition xmlns="http://www.w3.org/1999/xhtml"
                xmlns:f="jakarta.faces.core"
                xmlns:h="jakarta.faces.html"
                xmlns:ui="jakarta.faces.facelets"
                xmlns:p="primefaces">

    <ui:define name="left-nav">

        <h:panelGroup
                id="clientNavigation"
                layout="block"
                styleClass="nav-sub-lvl">

            <ul>

                <!-- Other menu links -->

                <h:panelGroup
                        id="secureShareLinkDiv"
                        layout="block"
                        rendered="#{
                            applicationAccessBean
                                .canRender('secureShareLink')
                        }">

                    <li>

                        <ui:insert name="secureShareLinkDef">

                            <h:commandLink
                                    id="secureShareLink"
                                    value="#{
                                        tBundle.SECURE_SHARE
                                    }"
                                    disabled="#{
                                        clientAccountNavBean
                                            .disabledStatusMap[
                                                'secureShareLink'
                                            ]
                                    }"
                                    styleClass="#{
                                        clientAccountNavBean
                                            .styleClassMap[
                                                'secureShareLink'
                                            ]
                                    }"
                                    actionListener="#{
                                        clientAccountNavBean.handleClick
                                    }"
                                    action="#{
                                        clientAccountNavBean.displayView
                                    }"
                                    target="SecureShareCDM"/>

                        </ui:insert>

                    </li>

                </h:panelGroup>

                <!-- Other menu links -->

            </ul>

        </h:panelGroup>

    </ui:define>

</ui:composition>

The old event is removed:

<!-- Remove from this component -->
<f:event
        type="preRenderComponent"
        listener="#{
            applicationAccessBean.checkComponentAccess
        }"/>

It is replaced by:

rendered="#{
    applicationAccessBean.canRender('secureShareLink')
}"

10. Access bean for 65 components

The important performance rule is:

> Do not make a database or service call every time canRender() is called.



JSF may evaluate rendered more than once.

Load all permissions once inside prepareAccess(), and make canRender() only perform an in-memory lookup.

package com.example.security;

import jakarta.faces.view.ViewScoped;
import jakarta.inject.Inject;
import jakarta.inject.Named;

import java.io.Serializable;
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

@Named
@ViewScoped
public class ApplicationAccessBean
        implements Serializable {

    private static final long serialVersionUID = 1L;

    @Inject
    private SecurityPolicyHandler securityPolicyHandler;

    private Map<String, Boolean> componentAccessMap;

    /**
     * Called once by f:viewAction on the initial request.
     *
     * Load all permissions here before Render Response.
     */
    public void prepareAccess() {

        Map<String, Boolean> access = new HashMap<>();

        access.put(
                "acctPerformanceLink",
                hasEntitlement("acctPerformanceLink")
        );

        access.put(
                "clientTradingProfileLink",
                hasEntitlement("clientTradingProfileLink")
        );

        access.put(
                "clientAcctPasswordManagementLink",
                hasEntitlement(
                        "clientAcctPasswordManagementLink"
                )
        );

        access.put(
                "dripDppManagementLink",
                hasEntitlement("dripDppManagementLink")
        );

        access.put(
                "secureShareLink",
                hasEntitlement("secureShareLink")
        );

        // Add the remaining secured component IDs here.

        componentAccessMap =
                Collections.unmodifiableMap(access);
    }

    private boolean hasEntitlement(String componentId) {

        return securityPolicyHandler
                .isEntitlementInRole(componentId);
    }

    /**
     * Called from rendered="#{...}".
     *
     * This method must only read the already-prepared map.
     * Do not access the database here.
     */
    public boolean canRender(String componentId) {

        if (componentAccessMap == null) {
            return false;
        }

        return Boolean.TRUE.equals(
                componentAccessMap.get(componentId)
        );
    }

    public Map<String, Boolean> getComponentAccessMap() {

        if (componentAccessMap == null) {
            return Collections.emptyMap();
        }

        return componentAccessMap;
    }
}

A better implementation, if your security service supports it, is to retrieve all user entitlements in one call:

public void prepareAccess() {

    componentAccessMap =
            Collections.unmodifiableMap(
                    securityPolicyHandler
                            .getComponentEntitlementsForCurrentUser()
            );
}

This is better than 65 separate calls:

Bad:
65 components × 1 database call
= 65 calls per page load

Preferred:

1 query/service call
        ↓
Return all user entitlements
        ↓
65 in-memory map lookups

11. Bean scope is important

Use:

@ViewScoped

with:

implements Serializable

Why?

Initial request
    ↓
prepareAccess() loads map
    ↓
Page is rendered
    ↓
User clicks SecureShare
    ↓
Postback restores same view-scoped bean
    ↓
Permission map is still available

If you use @RequestScoped with:

onPostback="false"

then this can happen:

Initial request → map loaded
Postback        → new bean created
viewAction      → does not run
map             → empty
all components  → rendered=false

Therefore, either:

Use @ViewScoped and onPostback="false", or

Use @RequestScoped and onPostback="true".


For your navigation view, @ViewScoped is usually more appropriate.

12. Preserve both actionListener and action

Your request is valid. They do different work.

actionListener

public void handleClick(ActionEvent event)

Purpose:

Identify clicked component

Update menu selection

Update CSS state

Log the event

Prepare values needed by the action


action

public String displayView()

Purpose:

Recheck authorization

Decide destination

Return navigation outcome


Complete Java code:

package com.example.navigation;

import jakarta.annotation.PostConstruct;
import jakarta.faces.application.FacesMessage;
import jakarta.faces.context.FacesContext;
import jakarta.faces.event.ActionEvent;
import jakarta.faces.view.ViewScoped;
import jakarta.inject.Inject;
import jakarta.inject.Named;

import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

@Named
@ViewScoped
public class ClientAccountNavBean
        implements Serializable {

    private static final long serialVersionUID = 1L;

    @Inject
    private SecurityPolicyHandler securityPolicyHandler;

    private Map<String, Boolean> disabledStatusMap;
    private Map<String, String> styleClassMap;

    private String selectedLink;

    @PostConstruct
    public void initializeNavigation() {

        disabledStatusMap = new HashMap<>();
        styleClassMap = new HashMap<>();

        disabledStatusMap.put(
                "secureShareLink",
                false
        );

        styleClassMap.put(
                "secureShareLink",
                "menu-link"
        );

        // Initialize the remaining menu links here.
    }

    /**
     * JSF calls this first.
     */
    public void handleClick(ActionEvent event) {

        selectedLink = event.getComponent().getId();

        styleClassMap.replaceAll(
                (componentId, currentClass) ->
                        "menu-link"
        );

        styleClassMap.put(
                selectedLink,
                "menu-link selected"
        );

        System.out.println(
                "Navigation listener: clicked "
                        + selectedLink
        );
    }

    /**
     * JSF calls this after handleClick().
     */
    public String displayView() {

        System.out.println(
                "Navigation action: selected "
                        + selectedLink
        );

        if ("secureShareLink".equals(selectedLink)) {

            boolean authorized =
                    securityPolicyHandler
                            .isEntitlementInRole(
                                    "secureShareLink"
                            );

            if (!authorized) {

                FacesContext.getCurrentInstance()
                        .addMessage(
                                null,
                                new FacesMessage(
                                        FacesMessage.SEVERITY_ERROR,
                                        "Access denied",
                                        "You are not authorized "
                                                + "to access SecureShare."
                                )
                        );

                return null;
            }

            return "/secureShare/secureShare.xhtml"
                    + "?faces-redirect=true";
        }

        return null;
    }

    public Map<String, Boolean>
            getDisabledStatusMap() {

        return disabledStatusMap;
    }

    public Map<String, String>
            getStyleClassMap() {

        return styleClassMap;
    }

    public String getSelectedLink() {
        return selectedLink;
    }
}

13. Exact click flow after the change

When the page first opens:

1. FacesServlet receives GET
2. JSF creates/restores the basic view
3. f:viewAction calls prepareAccess()
4. Component permission map is populated
5. JSF reaches Render Response
6. secureShareLinkDiv evaluates canRender()
7. If true, JSF renders the SecureShare link
8. If false, JSF skips the complete panel

When the user clicks SecureShare:

1. Browser clicks secureShareLink
2. h:commandLink submits the navigation form
3. target is set to SecureShareCDM
4. FacesServlet receives the POST
5. JSF restores the view
6. JSF identifies secureShareLink as the source
7. JSF queues ActionEvent
8. Process Validations runs
9. Update Model Values runs
10. Invoke Application starts
11. handleClick(ActionEvent) runs
12. selectedLink becomes secureShareLink
13. displayView() runs
14. Server checks authorization again
15. Action returns SecureShare navigation outcome
16. JSF redirects/renders SecureShare
17. Browser displays it in SecureShareCDM

For the same command, the expected order is:

actionListener
      ↓
action

14. Why check access twice?

The first check controls the UI:

rendered="#{
    applicationAccessBean.canRender('secureShareLink')
}"

This hides the menu from an unauthorized user.

The second check protects the operation:

securityPolicyHandler
    .isEntitlementInRole("secureShareLink");

Hiding a link is not security. A user might type the SecureShare URL directly.

Therefore:

UI check
   ↓
Better user experience

Action/filter security check
   ↓
Actual security

For maximum protection, also protect the SecureShare URL with Jakarta Security or a Servlet filter.

15. Do you need PrimeFaces changes?

For this link, no PrimeFaces-specific change is required.

Keep:

<h:commandLink>

It works with Faces 4 and PrimeFaces 15.

If you convert it to:

<p:commandLink>

you must add:

ajax="false"

because the navigation response goes to a named browser target:

<p:commandLink
        id="secureShareLink"
        value="#{tBundle.SECURE_SHARE}"
        disabled="#{
            clientAccountNavBean
                .disabledStatusMap['secureShareLink']
        }"
        styleClass="#{
            clientAccountNavBean
                .styleClassMap['secureShareLink']
        }"
        actionListener="#{
            clientAccountNavBean.handleClick
        }"
        action="#{
            clientAccountNavBean.displayView
        }"
        target="SecureShareCDM"
        ajax="false"/>

For minimum migration risk, use h:commandLink.

16. PrimeFaces 15 configuration

Use the Jakarta artifact:

<dependency>
    <groupId>org.primefaces</groupId>
    <artifactId>primefaces</artifactId>
    <version>15.0.0</version>
    <classifier>jakarta</classifier>
</dependency>

Do not manually load old jQuery:

<!-- Remove -->
<h:outputScript
        library="js"
        name="jquery/jquery-1.8.2.min.js"/>

Do not manually load the old JSF 2 resource:

<!-- Remove -->
<h:outputScript
        library="javax.faces"
        name="jsf.js"/>

Faces 4 and PrimeFaces will load the required Ajax resources.

17. Java package migration

Change:

javax.faces.*

to:

jakarta.faces.*

For this implementation:

import jakarta.faces.component.UIComponent;
import jakarta.faces.context.FacesContext;
import jakarta.faces.event.ActionEvent;
import jakarta.faces.view.ViewScoped;
import jakarta.inject.Inject;
import jakarta.inject.Named;

Replace old managed beans:

@ManagedBean

with CDI:

@Named

18. Migration pattern for the remaining 65 components

Old:

<h:panelGroup id="passwordManagementLinkDiv">

    ...

    <f:event
            type="preRenderComponent"
            listener="#{
                applicationAccessBean.checkComponentAccess
            }"/>

</h:panelGroup>

New:

<h:panelGroup
        id="passwordManagementLinkDiv"
        rendered="#{
            applicationAccessBean.canRender(
                'clientAcctPasswordManagementLink'
            )
        }">

    ...

</h:panelGroup>

Old:

<h:panelGroup id="dripDppManagementLinkDiv">

    ...

    <f:event
            type="preRenderComponent"
            listener="#{
                applicationAccessBean.checkComponentAccess
            }"/>

</h:panelGroup>

New:

<h:panelGroup
        id="dripDppManagementLinkDiv"
        rendered="#{
            applicationAccessBean.canRender(
                'dripDppManagementLink'
            )
        }">

    ...

</h:panelGroup>

The final architecture is:

BasePageTemplate.xhtml
    ↓
One f:metadata
    ↓
One f:viewAction
    ↓
prepareAccess()
    ↓
One permission map
    ↓
65 lightweight rendered lookups

This change removes the render-timing dependency, preserves both your action listener and action method, avoids repeated security calls during rendering, and works cleanly with Mojarra Faces 4 and PrimeFaces 15.