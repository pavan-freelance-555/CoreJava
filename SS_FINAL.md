Worked for 10s

Based on all three files you shared, you can fix SecureShare using only XHTML changes because ApplicationAccessBean already provides this boolean method:

public boolean isComponentEntitled(String componentId) {
    return getSecurityPolicyHandler()
            .isEntitlementInRole(componentId);
}

You do not need to add f:viewAction, prepareComponentAccess(), a new map, or change the Java code.

Make the following changes one at a time.

Step 1: Correct BasePageTemplate.xhtml

Because your existing method is:

public void checkRequestAccess(
        ComponentSystemEvent event)

keep using it as an event listener.

Remove the new f:metadata section that calls:

applicationAccessBean.checkRequestAccess
applicationAccessBean.prepareComponentAccess

prepareComponentAccess() does not exist in the Java class, and checkRequestAccess() expects a ComponentSystemEvent.

Use the original event:

<!DOCTYPE html>

<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:h="jakarta.faces.html"
      xmlns:f="jakarta.faces.core"
      xmlns:ui="jakarta.faces.facelets"
      xmlns:p="http://primefaces.org/ui"
      xmlns:c="jakarta.tags.core"
      xmlns:mot="http://www.midofficetools.bns.com/facelet/components">

    <!-- Keep the existing page-level access check -->
    <f:event
            type="preRenderView"
            listener="#{
                applicationAccessBean.checkRequestAccess
            }"/>

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

        <!-- Keep your existing head content -->

    </h:head>

    <h:body>

        <!-- Keep your existing body/template content -->

    </h:body>

</html>

This checks access to the complete page.

It is separate from SecureShare component visibility.

Step 2: Do not change the ClientAccountTemplate structure

Your ClientAccountTemplate.xhtml correctly uses:

<ui:composition
        xmlns="http://www.w3.org/1999/xhtml"
        xmlns:ui="jakarta.faces.facelets"
        xmlns:f="jakarta.faces.core"
        xmlns:h="jakarta.faces.html"
        xmlns:p="http://primefaces.org/ui"
        template="/template/BasePageTemplate.xhtml">

Keep this.

Also keep:

<ui:define name="left-nav">

The BasePageTemplate provides:

<ui:insert name="left-nav">

and the ClientAccountTemplate fills that location using:

<ui:define name="left-nav">

The effective flow is:

BasePageTemplate.xhtml
        ↓
Finds ui:insert name="left-nav"
        ↓
ClientAccountTemplate.xhtml supplies ui:define name="left-nav"
        ↓
Client account menu appears

Step 3: Change only the SecureShare panel

Your current SecureShare section is:

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

Make two changes:

1. Add rendered to secureShareLinkDiv.


2. Remove the preRenderComponent event from inside that panel.



The final SecureShare section should be:

<h:panelGroup
        layout="block"
        id="secureShareLinkDiv"
        rendered="#{
            applicationAccessBean
                .isComponentEntitled(
                    'secureShareLinkDiv'
                )
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
                    target="SecureShareCDM"
                    action="#{
                        clientAccountNavBean.displayView
                    }"/>

        </ui:insert>

    </li>

</h:panelGroup>

These parts remain unchanged:

actionListener="#{clientAccountNavBean.handleClick}"
action="#{clientAccountNavBean.displayView}"
target="SecureShareCDM"
disabled="#{...}"
styleClass="#{...}"

Only this is removed:

<f:event
        type="preRenderComponent"
        listener="#{
            applicationAccessBean.checkComponentAccess
        }"/>

And this is added:

rendered="#{
    applicationAccessBean
        .isComponentEntitled('secureShareLinkDiv')
}"

Step 4: Confirm the entitlement key

Your Java method previously used:

UIComponent component = event.getComponent();

String componentId =
        component.getClientId();

getSecurityPolicyHandler()
        .isEntitlementInRole(componentId);

Because the event was inside:

<h:panelGroup id="secureShareLinkDiv">

the old component ID was probably:

secureShareLinkDiv

This is especially likely because your enclosing form has:

prependId="false"

However, confirm the configured name in:

componentSecurityPolicy.xml

Search for:

secureShareLinkDiv

If the configuration contains:

secureShareLinkDiv

then use:

rendered="#{
    applicationAccessBean
        .isComponentEntitled('secureShareLinkDiv')
}"

If the security configuration instead contains:

secureShareLink

then use:

rendered="#{
    applicationAccessBean
        .isComponentEntitled('secureShareLink')
}"

The string must match exactly, including uppercase and lowercase characters.

Step 5: What happens during initial page rendering?

After these changes:

Browser requests ClientAccount page
        ↓
BasePageTemplate.xhtml loads
        ↓
preRenderView event executes
        ↓
checkRequestAccess(ComponentSystemEvent) executes
        ↓
Whole-page access is checked
        ↓
ClientAccountTemplate supplies left-nav
        ↓
JSF reaches secureShareLinkDiv
        ↓
JSF evaluates rendered
        ↓
isComponentEntitled("secureShareLinkDiv") executes
        ↓
SecurityPolicyHandler checks entitlement

If the user is entitled:

isComponentEntitled(...) = true
        ↓
secureShareLinkDiv is encoded
        ↓
SecureShare appears

If the user is not entitled:

isComponentEntitled(...) = false
        ↓
JSF skips secureShareLinkDiv
        ↓
SecureShare does not appear

This avoids the Faces 4 timing issue.

The old implementation did this:

Start component rendering
        ↓
preRenderComponent event
        ↓
setRendered(false)

The new implementation does this:

Ask for rendered value
        ↓
Permission returns false
        ↓
Do not start rendering

Step 6: What happens when you click SecureShare?

Assume the user is entitled and the link is enabled.

Browser submits the form

h:commandLink performs a normal JSF form submission.

Click SecureShare
        ↓
Submit enclosing h:form
        ↓
POST request goes to FacesServlet

This is not PrimeFaces Ajax. Therefore, no ajax="false" is required.

JSF restores the view

Restore View
        ↓
Restore ClientAccount component tree

JSF identifies the command link

During Apply Request Values:

secureShareLink decoded
        ↓
ActionEvent queued

Listener executes

During Invoke Application:

actionListener="#{
    clientAccountNavBean.handleClick
}"

JSF calls:

handleClick(ActionEvent event)

This normally identifies the clicked link and prepares the selected navigation state.

Action executes

After the listener:

action="#{
    clientAccountNavBean.displayView
}"

JSF calls:

displayView()

The method returns the destination for SecureShare.

The order is:

handleClick()
        ↓
displayView()

Target window is used

You have:

target="SecureShareCDM"

Therefore:

Navigation result
        ↓
Displayed in a browser window/tab named SecureShareCDM

The complete click flow is:

User clicks SecureShare
        ↓
h:commandLink submits form
        ↓
FacesServlet receives postback
        ↓
Restore View
        ↓
Apply Request Values
        ↓
ActionEvent queued
        ↓
Process Validations
        ↓
Update Model Values
        ↓
handleClick(ActionEvent)
        ↓
displayView()
        ↓
Navigation outcome
        ↓
Response opens in SecureShareCDM

Step 7: No PrimeFaces 15 change is required for this link

You currently use:

<h:commandLink>

This is a Jakarta Faces component. It works with:

Jakarta Faces 4
Mojarra 4
PrimeFaces 15

Do not change it to p:commandLink unless you require a PrimeFaces feature.

Keeping h:commandLink preserves the existing behavior:

Non-Ajax form submission

actionListener followed by action

Named target window

Existing CSS styling


Your namespace is already appropriate:

xmlns:p="http://primefaces.org/ui"

Step 8: Test with a hard-coded value first

If you want to isolate whether the problem is security or rendering, temporarily test:

<h:panelGroup
        layout="block"
        id="secureShareLinkDiv"
        rendered="#{true}">

Expected:

SecureShare always appears

Then test:

rendered="#{false}"

Expected:

SecureShare disappears

Finally restore:

rendered="#{
    applicationAccessBean
        .isComponentEntitled('secureShareLinkDiv')
}"

This verifies three different layers:

Test	What it proves

rendered="#{true}"	XHTML/component renders correctly
rendered="#{false}"	Faces 4 respects the rendered attribute
Entitlement method	Security configuration returns correct result


Step 9: Add temporary UI diagnostics if needed

To confirm the security result without changing Java, temporarily add:

<h:outputText
        value="SecureShare entitlement: #{
            applicationAccessBean
                .isComponentEntitled(
                    'secureShareLinkDiv'
                )
        }"/>

This will display:

SecureShare entitlement: true

or:

SecureShare entitlement: false

Remove this output after testing.

Do not place it inside secureShareLinkDiv, because if the result is false, the entire parent will be hidden.

Place it immediately before the panel:

<h:outputText
        value="SecureShare entitlement: #{
            applicationAccessBean
                .isComponentEntitled(
                    'secureShareLinkDiv'
                )
        }"/>

<h:panelGroup
        id="secureShareLinkDiv"
        layout="block"
        rendered="#{
            applicationAccessBean
                .isComponentEntitled(
                    'secureShareLinkDiv'
                )
        }">

    ...

</h:panelGroup>

Step 10: Do not convert all existing events yet

Your checkComponentAccess() method contains special dropdown processing for:

HtmlSelectOneMenu
HtmlSelectManyMenu
UISelectItems

For those components, it:

Reads all dropdown options

Reads role entitlements

Filters unauthorized values

Creates a new option list

Selects a default value

Potentially hides the dropdown

Updates related session UI beans


A simple rendered expression cannot replace that processing.

For now:

Convert SecureShare and similar normal links.

Keep existing dropdown events until they are migrated separately.

Do not remove every preRenderComponent occurrence globally.


Final UI-only changes

BasePageTemplate.xhtml

Keep:

<f:event
        type="preRenderView"
        listener="#{
            applicationAccessBean.checkRequestAccess
        }"/>

Remove the newly added f:metadata/f:viewAction code.

ClientAccountTemplate.xhtml

Change SecureShare to:

<h:panelGroup
        layout="block"
        id="secureShareLinkDiv"
        rendered="#{
            applicationAccessBean
                .isComponentEntitled(
                    'secureShareLinkDiv'
                )
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
                    action="#{
                        clientAccountNavBean.displayView
                    }"
                    target="SecureShareCDM"/>

        </ui:insert>

    </li>

</h:panelGroup>

That is the complete UI-only SecureShare migration based on your existing Java implementation.