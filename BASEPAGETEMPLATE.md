Worked for 14s

Now that I can see your actual BasePageTemplate.xhtml, the placement is clear.

Your file does not use <f:view> as the root. It uses regular <html> as the root, which is valid in Faces 4. Do not add <f:view> only for this change.

Your current code begins approximately like this:

<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="jakarta.faces.html"
      xmlns:f="jakarta.faces.core"
      xmlns:ui="jakarta.faces.facelets"
      xmlns:p="http://primefaces.org/ui"
      xmlns:c="jakarta.tags.core"
      xmlns:mot="http://www.midofficetools.bns.com/facelet/components">

    <f:event
            type="preRenderView"
            listener="#{
                applicationAccessBean.checkRequestAccess
            }"/>

    ...
</html>

Replace that page-level f:event with f:metadata and f:viewAction.

1. Exact change in BasePageTemplate.xhtml

Remove this from BasePageTemplate.xhtml

<f:event
        type="preRenderView"
        listener="#{
            applicationAccessBean.checkRequestAccess
        }"/>

Add this in the same location

It should be the first meaningful child immediately after the opening <html> tag:

<f:metadata>

    <f:viewAction
            action="#{
                applicationAccessBean.preparePageAccess
            }"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>

    <ui:insert name="metadata"/>

</f:metadata>

The beginning of your actual file should become:

<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="jakarta.faces.html"
      xmlns:f="jakarta.faces.core"
      xmlns:ui="jakarta.faces.facelets"
      xmlns:p="http://primefaces.org/ui"
      xmlns:c="jakarta.tags.core"
      xmlns:mot="http://www.midofficetools.bns.com/facelet/components"
      lang="en">

    <!-- Page-level metadata must appear before h:head -->
    <f:metadata>

        <!--
            Runs early and prepares:
            1. Whole-page access
            2. Component visibility
        -->
        <f:viewAction
                action="#{
                    applicationAccessBean.preparePageAccess
                }"
                phase="APPLY_REQUEST_VALUES"
                onPostback="false"/>

        <!-- Allows child pages to add their own metadata -->
        <ui:insert name="metadata"/>

    </f:metadata>

    <ui:remove>
        <ui:debug/>
    </ui:remove>

    <f:loadBundle
            basename="application"
            var="tBundle"/>

    <f:loadBundle
            basename="application_gif"
            var="gBundle"/>

    <h:head>
        ...
    </h:head>

    <h:body>
        ...
    </h:body>

</html>

The position is:

<html opening tag
        ↓
f:metadata
        ↓
f:viewAction
        ↓
ui:remove / loadBundle
        ↓
h:head
        ↓
h:body

Do not place f:metadata inside h:head, h:body, h:form or h:panelGroup.

2. There are two different f:event uses

Your application has two different event types.

Event A: preRenderView in BasePageTemplate.xhtml

Your screenshot shows:

<f:event
        type="preRenderView"
        listener="#{
            applicationAccessBean.checkRequestAccess
        }"/>

This is a whole-page event.

Replace it with:

<f:metadata>
    <f:viewAction
            action="#{
                applicationAccessBean.preparePageAccess
            }"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>
</f:metadata>

Event B: preRenderComponent beside secured components

SecureShare currently has:

<f:event
        type="preRenderComponent"
        listener="#{
            applicationAccessBean.checkComponentAccess
        }"/>

This is a component-level event.

Remove it from the SecureShare panel and replace it with a rendered expression:

<h:panelGroup
        id="secureShareLinkDiv"
        layout="block"
        rendered="#{
            applicationAccessBean
                .canRender('secureShareLink')
        }">

Therefore:

preRenderView
    → replaced by one f:viewAction in BasePageTemplate.xhtml

preRenderComponent
    → replaced by rendered on each secured component

3. Why use preparePageAccess()?

Your current system performs two kinds of checks:

checkRequestAccess()
    → Can the user access the complete page?

checkComponentAccess()
    → Can the user see an individual component?

The new method prepares both before rendering begins:

public String preparePageAccess() {

    // 1. Check complete-page access

    // 2. If page is allowed, load all component permissions

    // 3. Return null so JSF continues to render the page
}

The lifecycle becomes:

Request enters Faces
        ↓
Restore/build basic view
        ↓
APPLY_REQUEST_VALUES
        ↓
preparePageAccess()
        ↓
Check page access
        ↓
Load all component permissions
        ↓
Continue lifecycle
        ↓
Render Response
        ↓
Each component evaluates rendered

This avoids changing component visibility after JSF has already started rendering it.

4. Complete ApplicationAccessBean

Here is the recommended Java structure:

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

    private Map<String, Boolean> componentAccessMap =
            Collections.emptyMap();

    /**
     * Called once from f:viewAction on the initial page request.
     */
    public String preparePageAccess() {

        /*
         * Step 1:
         * Check whether the user can access the entire page.
         */
        boolean requestAllowed =
                securityPolicyHandler.isRequestAllowed();

        if (!requestAllowed) {

            /*
             * Returning a navigation outcome stops the user
             * from continuing to the requested page.
             */
            return "/access-denied.xhtml"
                    + "?faces-redirect=true";
        }

        /*
         * Step 2:
         * The user can access the page.
         * Load visibility permissions for the components.
         */
        prepareComponentAccess();

        /*
         * Step 3:
         * null means continue displaying the current page.
         */
        return null;
    }

    private void prepareComponentAccess() {

        Map<String, Boolean> access =
                new HashMap<>();

        access.put(
                "acctPerformanceLink",
                checkEntitlement("acctPerformanceLink")
        );

        access.put(
                "clientTradingProfileLink",
                checkEntitlement("clientTradingProfileLink")
        );

        access.put(
                "clientAcctPasswordManagementLink",
                checkEntitlement(
                        "clientAcctPasswordManagementLink"
                )
        );

        access.put(
                "dripDppManagementLink",
                checkEntitlement("dripDppManagementLink")
        );

        access.put(
                "secureShareLink",
                checkEntitlement("secureShareLink")
        );

        // Add the remaining secured component keys here.

        componentAccessMap =
                Collections.unmodifiableMap(access);
    }

    private boolean checkEntitlement(String componentId) {

        return securityPolicyHandler
                .isEntitlementInRole(componentId);
    }

    /**
     * Called from the XHTML rendered attribute.
     *
     * This must only perform an in-memory lookup.
     * Do not query the database here.
     */
    public boolean canRender(String componentId) {

        return Boolean.TRUE.equals(
                componentAccessMap.get(componentId)
        );
    }

    public Map<String, Boolean>
            getComponentAccessMap() {

        return componentAccessMap;
    }
}

You will need to adapt:

securityPolicyHandler.isRequestAllowed()

to the actual whole-request access method already used by your existing checkRequestAccess() implementation.

The important design is:

preparePageAccess()
    ├── existing checkRequestAccess logic
    └── load component permission map

5. If you do not want to combine the two access methods

You previously said that you want to keep the command’s listener and action separate. That is fully supported and unrelated to this page preparation.

However, if you also want checkRequestAccess() and prepareComponentAccess() to remain separate, use two f:viewAction tags:

<f:metadata>

    <f:viewAction
            action="#{
                applicationAccessBean.checkRequestAccess
            }"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>

    <f:viewAction
            action="#{
                applicationAccessBean.prepareComponentAccess
            }"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>

    <ui:insert name="metadata"/>

</f:metadata>

Java:

public String checkRequestAccess() {

    if (!securityPolicyHandler.isRequestAllowed()) {
        return "/access-denied.xhtml"
                + "?faces-redirect=true";
    }

    return null;
}

public void prepareComponentAccess() {

    Map<String, Boolean> access =
            new HashMap<>();

    access.put(
            "secureShareLink",
            securityPolicyHandler.isEntitlementInRole(
                    "secureShareLink"
            )
    );

    // Other components...

    componentAccessMap =
            Collections.unmodifiableMap(access);
}

I recommend the single preparePageAccess() method because it guarantees:

Check page access first
        ↓
Only if allowed, load component access

But this does not affect your requirement to preserve the SecureShare actionListener and action.

6. Exact SecureShare conversion

Current version

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

Faces 4 version

<h:panelGroup
        layout="block"
        id="secureShareLinkDiv"
        rendered="#{
            applicationAccessBean
                .canRender('secureShareLink')
        }">

    <li>

        <ui:insert name="secureShareLinkDef">

            <h:commandLink
                    id="secureShareLink"
                    value="#{tBundle.SECURE_SHARE}"
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
                    target="SecureShareCDM"
                    action="#{
                        clientAccountNavBean.displayView
                    }"/>

        </ui:insert>

    </li>

</h:panelGroup>

Notice that these remain unchanged:

actionListener="#{clientAccountNavBean.handleClick}"
action="#{clientAccountNavBean.displayView}"
target="SecureShareCDM"

Only this was removed:

<f:event type="preRenderComponent" .../>

And this was added:

rendered="#{
    applicationAccessBean.canRender('secureShareLink')
}"

7. Keep actionListener and action separately

Your Java methods can remain separate.

import jakarta.faces.event.ActionEvent;

public void handleClick(ActionEvent event) {

    selectedLink = event.getComponent().getId();

    logger.debug(
            "Clicked navigation link: {}",
            selectedLink
    );

    // Update selected menu styling or navigation state.
}

Then:

public String displayView() {

    if ("secureShareLink".equals(selectedLink)) {

        /*
         * Recheck security here.
         *
         * Hiding the menu link is not enough security.
         */
        boolean allowed =
                securityPolicyHandler.isEntitlementInRole(
                        "secureShareLink"
                );

        if (!allowed) {
            return "/access-denied.xhtml"
                    + "?faces-redirect=true";
        }

        return "/secureShare/secureShare.xhtml"
                + "?faces-redirect=true";
    }

    return null;
}

The execution order is:

User clicks SecureShare
        ↓
handleClick(ActionEvent)
        ↓
selectedLink = secureShareLink
        ↓
displayView()
        ↓
Check authorization again
        ↓
Return destination

8. One important issue visible in your template

Your screenshot shows:

<c:if test="#{not empty sessionScope.SESSION_USER}">

Be careful with JSTL c:if around JSF components.

c:if controls whether components are added while the view is being built. It does not behave like the JSF rendered attribute.

A safer Faces version is:

<h:panelGroup
        layout="block"
        rendered="#{
            not empty sessionScope.SESSION_USER
        }">

    <mot:dashboard .../>

    <h:panelGroup
            id="fundsdashboardid"
            rendered="#{
                tBundle.FUNDS_DASHBOARD_ENABLE
            }">

        <mot:fundsDashboard .../>

    </h:panelGroup>

</h:panelGroup>

Instead of:

<c:if test="#{not empty sessionScope.SESSION_USER}">
    ...
</c:if>

Why?

c:if
    → Changes component-tree construction

rendered
    → Keeps component tree stable and controls encoding

Stable component trees are safer for Faces postbacks, Ajax updates and partial-state saving.

This may be particularly important during your Mojarra 2 to Mojarra 4 migration.

9. PrimeFaces 15 changes for this template

Your current namespace is:

xmlns:p="http://primefaces.org/ui"

You can keep it. It is the conventional PrimeFaces tag-library namespace.

For your SecureShare link, keep:

<h:commandLink>

No PrimeFaces change is required.

If you later change it to:

<p:commandLink>

use:

ajax="false"

because you use:

target="SecureShareCDM"

Example:

<p:commandLink
        id="secureShareLink"
        value="#{tBundle.SECURE_SHARE}"
        actionListener="#{
            clientAccountNavBean.handleClick
        }"
        action="#{
            clientAccountNavBean.displayView
        }"
        target="SecureShareCDM"
        ajax="false"/>

10. One more cleanup visible in the screenshot

Your template loads:

<h:outputScript
        library="jakarta.faces"
        name="faces.js"/>

Faces/PrimeFaces normally loads the required Faces JavaScript automatically when an Ajax-enabled component needs it.

You can usually remove the manual resource declaration:

<h:outputScript
        library="jakarta.faces"
        name="faces.js"/>

Also avoid manually loading an old jQuery version. Your screenshot appears to load:

jquery/select2.full.min.js

That is not jQuery itself, but confirm that another file is not loading:

jquery-1.8.2.min.js

PrimeFaces 15 provides its compatible jQuery version. Old jQuery can break PrimeFaces widgets.

Exact change summary

In your actual BasePageTemplate.xhtml:

1. Keep the root <html>. Do not add <f:view>.


2. Remove:



<f:event
        type="preRenderView"
        listener="#{
            applicationAccessBean.checkRequestAccess
        }"/>

3. Add immediately after the opening <html>:



<f:metadata>
    <f:viewAction
            action="#{
                applicationAccessBean.preparePageAccess
            }"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>

    <ui:insert name="metadata"/>
</f:metadata>

4. In every secured component file, remove:



<f:event
        type="preRenderComponent"
        listener="#{
            applicationAccessBean.checkComponentAccess
        }"/>

5. Add to its enclosing panel:



rendered="#{
    applicationAccessBean.canRender('permissionKey')
}"

6. Keep SecureShare’s actionListener and action unchanged.


7. Prefer JSF rendered over JSTL c:if for components that must survive postbacks or Ajax requests.


