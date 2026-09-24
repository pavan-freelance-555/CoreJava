Worked for 18s

When you click SecureShare, this is a normal full JSF form submission—not an Ajax request.

Your current code is approximately:

<h:panelGroup layout="block" id="secureShareLinkDiv">

    <li>
        <ui:insert name="secureShareLinkDef">

            <h:commandLink
                    id="secureShareLink"
                    value="#{tBundle.SECURE_SHARE}"
                    disabled="#{clientAccountNavBean.disabledStatusMap['secureShareLink']}"
                    styleClass="#{clientAccountNavBean.styleClassMap['secureShareLink']}"
                    actionListener="#{clientAccountNavBean.handleClick}"
                    target="SecureShareCDM"
                    action="#{clientAccountNavBean.displayView}"/>

        </ui:insert>
    </li>

    <f:event
            type="preRenderComponent"
            listener="#{applicationAccessBean.checkComponentAccess}"/>

</h:panelGroup>

1. What happens while initially displaying the left panel?

Before you click anything, JSF builds the page.

Step 1: JSF creates the parent component

This creates an HtmlPanelGroup:

<h:panelGroup
        layout="block"
        id="secureShareLinkDiv">

It normally produces:

<div id="secureShareLinkDiv">

Step 2: JSF processes ui:insert

<ui:insert name="secureShareLinkDef">

This is a template insertion point.

If a child page contains:

<ui:define name="secureShareLinkDef">
    <!-- Different content -->
</ui:define>

then that custom content replaces your default SecureShare command link.

If no child page provides that definition, JSF uses the default content:

<h:commandLink id="secureShareLink" .../>

Step 3: JSF evaluates the link label

value="#{tBundle.SECURE_SHARE}"

JSF reads the resource bundle.

For example:

SECURE_SHARE=SecureShare

The link displays:

SecureShare

Step 4: JSF checks whether the link is disabled

disabled="#{
    clientAccountNavBean.disabledStatusMap['secureShareLink']
}"

Suppose the map contains:

disabledStatusMap.put("secureShareLink", false);

Then:

disabled = false

The user can click the link.

If it contains:

disabledStatusMap.put("secureShareLink", true);

then the link is displayed but cannot be clicked.

Step 5: JSF gets the CSS class

styleClass="#{
    clientAccountNavBean.styleClassMap['secureShareLink']
}"

For example:

styleClassMap.put("secureShareLink", "active-menu-link");

JSF produces something similar to:

<a class="active-menu-link">
    SecureShare
</a>

This is why the SecureShare item appears with the red styling in your screenshot.

Step 6: preRenderComponent runs

Because this event is inside secureShareLinkDiv:

<f:event
        type="preRenderComponent"
        listener="#{applicationAccessBean.checkComponentAccess}"/>

the event is attached to the enclosing panel group:

secureShareLinkDiv

The event object points to that component:

public void checkComponentAccess(
        ComponentSystemEvent event) {

    UIComponent component = event.getComponent();
}

Here, component is normally:

HtmlPanelGroup secureShareLinkDiv

It is not necessarily the inner secureShareLink.

That is why your debugger showed something like:

component = HtmlPanelGroup

The intended logic is:

Does user have SecureShare permission?
   ├── Yes → render panel and link
   └── No  → component.setRendered(false)

However, this event happens during the Render Response phase, after JSF has already entered the parent component’s rendering operation. That makes changing rendered from the component’s own preRenderComponent event fragile in Faces 4.


---

2. What happens when you click SecureShare?

Assume:

disabled = false

and the link is visible.

Step 1: Browser executes the link JavaScript

h:commandLink does not render as a simple URL link.

JSF generates JavaScript that submits the surrounding form.

Conceptually, the generated HTML works like:

<a href="#"
   onclick="submitTheJsfForm();">
    SecureShare
</a>

Therefore, the command link must be inside an h:form.

Your base template appears to have:

<h:form prependId="false">

If there is no enclosing form, the command link will not work correctly.

Step 2: Target window is selected

Your link contains:

target="SecureShareCDM"

That means the response should be displayed in a browser window or tab named:

SecureShareCDM

Browser behavior is generally:

First click
   ↓
Open new window/tab named SecureShareCDM

Later clicks may reuse the same named window:

Another click
   ↓
Find existing SecureShareCDM window
   ↓
Load response in that window

This is different from:

target="_blank"

_blank generally requests a new browsing context each time, while a fixed name such as SecureShareCDM can be reused.

Step 3: The form is submitted

The browser sends an HTTP POST:

Browser
   ↓
POST current-page.xhtml
   ↓
FacesServlet

This is a full JSF submit because there is no:

<f:ajax/>

and no PrimeFaces Ajax behavior.

Step 4: JSF restores the view

JSF executes:

Restore View

It restores the server-side component tree containing:

left navigation form
    ↓
secureShareLinkDiv
    ↓
secureShareLink

Step 5: JSF decodes the request

During:

Apply Request Values

JSF determines which command component was clicked.

The request contains information indicating:

secureShareLink was clicked

JSF queues an ActionEvent for that component.

Step 6: Validation and model update run

Unless immediate="true" is used, JSF continues:

Process Validations
        ↓
Update Model Values

Because the left navigation form may contain only menu links, there might be nothing significant to validate.

But if the same form contains required input fields, those validations can prevent the SecureShare action from executing.

That is one reason navigation menus should usually have their own small form.

Step 7: actionListener executes

During Invoke Application, JSF calls:

actionListener="#{clientAccountNavBean.handleClick}"

The method should normally look like:

import jakarta.faces.event.ActionEvent;

public void handleClick(ActionEvent event) {

    String clickedComponentId =
            event.getComponent().getId();

    logger.debug(
            "Clicked component: {}",
            clickedComponentId
    );

    // Expected: secureShareLink
}

At this point:

clickedComponentId = secureShareLink

The action listener is normally used for side effects, such as:

Recording which menu was clicked

Setting the selected menu

Setting temporary bean values

Logging

Preparing data needed by the action


It should normally return void.

Step 8: The action method executes

After the action listener, JSF calls:

action="#{clientAccountNavBean.displayView}"

A typical method is:

public String displayView() {

    return "/secureShare/secureShare.xhtml";
}

Or:

public String displayView() {

    return "/secureShare/secureShare.xhtml"
            + "?faces-redirect=true";
}

The return value is the navigation outcome.

For example:

displayView()
      ↓
Returns /secureShare/secureShare.xhtml
      ↓
JSF navigates to SecureShare page

Because the command link has:

target="SecureShareCDM"

the resulting page is displayed in the SecureShareCDM browser window/tab.

Complete click flow

User clicks SecureShare
        ↓
Browser submits enclosing JSF form
        ↓
Response target = SecureShareCDM
        ↓
FacesServlet receives POST
        ↓
Restore View
        ↓
Apply Request Values
        ↓
secureShareLink detected
        ↓
ActionEvent queued
        ↓
Process Validations
        ↓
Update Model Values
        ↓
handleClick(ActionEvent) executes
        ↓
displayView() executes
        ↓
Navigation outcome returned
        ↓
SecureShare page rendered
        ↓
Response shown in SecureShareCDM window

3. Important difference between the two methods

Your link calls two methods:

actionListener="#{clientAccountNavBean.handleClick}"
action="#{clientAccountNavBean.displayView}"

Their responsibilities should be different.

public void handleClick(ActionEvent event) {
    // Side effect and preparation.
}

public String displayView() {
    // Decide where to navigate.
    return "/secureShare/secureShare.xhtml";
}

If handleClick() only determines the destination used by displayView(), you can simplify everything into one action method.

For example:

public String openSecureShare() {

    logger.debug("Opening SecureShare");

    if (!securityPolicyHandler
            .isEntitlementInRole("secureShareLink")) {

        return "/access-denied.xhtml"
                + "?faces-redirect=true";
    }

    return "/secureShare/secureShare.xhtml"
            + "?faces-redirect=true";
}

XHTML:

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
        target="SecureShareCDM"
        action="#{clientAccountNavBean.openSecureShare}"/>

This is easier to understand:

Click
  ↓
openSecureShare()
  ↓
Authorize
  ↓
Return destination

4. Recommended Faces 4 access-control change

Remove this:

<f:event
        type="preRenderComponent"
        listener="#{applicationAccessBean.checkComponentAccess}"/>

Do not wait until component rendering to decide whether SecureShare should be displayed.

Instead, calculate the permission before Render Response and bind it directly to the panel.

Bean

package com.example.navigation;

import jakarta.annotation.PostConstruct;
import jakarta.faces.view.ViewScoped;
import jakarta.inject.Inject;
import jakarta.inject.Named;

import java.io.Serializable;
import java.util.HashMap;
import java.util.Map;

@Named
@ViewScoped
public class ApplicationAccessBean implements Serializable {

    private static final long serialVersionUID = 1L;

    @Inject
    private SecurityPolicyHandler securityPolicyHandler;

    private Map<String, Boolean> componentAccessMap;

    @PostConstruct
    public void init() {
        componentAccessMap = new HashMap<>();
    }

    public void loadComponentAccess() {

        componentAccessMap.put(
                "secureShareLink",
                securityPolicyHandler.isEntitlementInRole(
                        "secureShareLink"
                )
        );
    }

    public Map<String, Boolean> getComponentAccessMap() {
        return componentAccessMap;
    }
}

Page metadata

Place this near the top-level view:

<f:metadata>

    <f:viewAction
            action="#{applicationAccessBean.loadComponentAccess}"
            phase="APPLY_REQUEST_VALUES"
            onPostback="false"/>

</f:metadata>

This gives the following order:

Restore view
     ↓
Load access permissions
     ↓
Render Response starts
     ↓
Check rendered expression
     ↓
Render or skip SecureShare

Updated SecureShare XHTML

<h:panelGroup
        id="secureShareLinkDiv"
        layout="block"
        rendered="#{
            applicationAccessBean
                .componentAccessMap['secureShareLink']
        }">

    <ul>
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
                        target="SecureShareCDM"
                        action="#{
                            clientAccountNavBean.openSecureShare
                        }"/>

            </ui:insert>

        </li>
    </ul>

</h:panelGroup>

Be careful with the surrounding HTML. In your screenshot, the outer <ul> already appears to exist. If it already exists, do not add another <ul> around this block:

<ul>
    ...
    <h:panelGroup id="secureShareLinkDiv">
        <li>...</li>
    </h:panelGroup>
    ...
</ul>

5. PrimeFaces 15 version

You do not have to convert every h:commandLink to PrimeFaces. h:commandLink works correctly with Jakarta Faces 4.

If you want to use PrimeFaces 15, use:

xmlns:p="primefaces"

Then:

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
        action="#{clientAccountNavBean.openSecureShare}"
        target="SecureShareCDM"
        ajax="false"/>

The important setting is:

ajax="false"

PrimeFaces command components use Ajax by default. For navigation into a target browser window, use a normal full form submission:

ajax="false"

Recommended PrimeFaces code:

<h:panelGroup
        id="secureShareLinkDiv"
        layout="block"
        rendered="#{
            applicationAccessBean
                .componentAccessMap['secureShareLink']
        }">

    <li>
        <ui:insert name="secureShareLinkDef">

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
                    action="#{
                        clientAccountNavBean.openSecureShare
                    }"
                    target="SecureShareCDM"
                    ajax="false"/>

        </ui:insert>
    </li>

</h:panelGroup>

6. Java migration changes

Your action listener must use:

import jakarta.faces.event.ActionEvent;

not:

import javax.faces.event.ActionEvent;

Your system event, if still used elsewhere, must use:

import jakarta.faces.event.ComponentSystemEvent;

Your components must use:

import jakarta.faces.component.UIComponent;

Your beans should use CDI:

import jakarta.inject.Named;
import jakarta.faces.view.ViewScoped;

Example:

@Named
@ViewScoped
public class ClientAccountNavBean
        implements Serializable {
}

Do not use:

javax.faces.bean.ManagedBean

or:

javax.faces.bean.ViewScoped

7. Important security correction

This:

rendered="false"

only hides the link.

It does not secure the SecureShare functionality.

A user could potentially call the target URL directly:

/secureShare/secureShare.xhtml

Therefore, check authorization in two places:

UI check
   ↓
Hide SecureShare link

and:

Server-side check
   ↓
Prevent unauthorized action/page access

For example:

public String openSecureShare() {

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

For stronger protection, use a Servlet filter or Jakarta Security constraint on the SecureShare URL.

Final recommendation

For Faces 4 and PrimeFaces 15, use this design:

f:viewAction
   ↓
Load SecureShare entitlement early
   ↓
rendered expression decides visibility
   ↓
User clicks link
   ↓
p:commandLink ajax=false
   ↓
Single action method checks access again
   ↓
Open SecureShare in SecureShareCDM window

The key changes are:

Remove preRenderComponent for visibility control.

Bind the parent panel’s rendered property to a boolean or permission map.

Prefer one action method instead of separate actionListener and action when possible.

If using p:commandLink, set ajax="false" for the target window.

Change all javax.faces.* imports to jakarta.faces.*.

Enforce access again on the server; hiding the menu alone is not security.