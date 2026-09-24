The main difference is only when SecureShare entitlement is checked.

The page-level access flow and the click flow remain almost the same.

Old JSF 2 flow

Your old SecureShare code was:

<h:panelGroup
        id="secureShareLinkDiv"
        layout="block">

    <li>
        <h:commandLink
                id="secureShareLink"
                actionListener="#{
                    clientAccountNavBean.handleClick
                }"
                action="#{
                    clientAccountNavBean.displayView
                }"
                target="SecureShareCDM"/>
    </li>

    <f:event
            type="preRenderComponent"
            listener="#{
                applicationAccessBean.checkComponentAccess
            }"/>

</h:panelGroup>

The flow was:

1. JSF creates secureShareLinkDiv
        ↓
2. secureShareLinkDiv starts Render Response processing
        ↓
3. Initially component.rendered = true
        ↓
4. preRenderComponent event fires
        ↓
5. checkComponentAccess(event) executes
        ↓
6. Java gets the component from the event
        ↓
7. Java gets component.getClientId()
        ↓
8. Java checks isEntitlementInRole(componentId)
        ↓
9. If unauthorized:
   component.setRendered(false)
        ↓
10. Old Mojarra checks rendered again
        ↓
11. Component is skipped

The key part is:

component.setRendered(false);

Your listener changed the component after JSF entered that component’s rendering process.

Why it appeared to work in JSF 2

Mojarra 2 behavior apparently checked the rendered state again after the event or before the final encoding operation:

First rendered check = true
        ↓
preRenderComponent event
        ↓
setRendered(false)
        ↓
Second rendered check = false
        ↓
Skip component

Your application therefore depended on this implementation behavior.

The preRenderComponent event was still late, but Mojarra 2’s execution happened to support your pattern.

New Faces 4 flow

The new code is:

<h:panelGroup
        id="secureShareLinkDiv"
        layout="block"
        rendered="#{
            applicationAccessBean
                .isComponentEntitled(
                    'secureShareLinkDiv'
                )
        }">

    <li>
        <h:commandLink
                id="secureShareLink"
                actionListener="#{
                    clientAccountNavBean.handleClick
                }"
                action="#{
                    clientAccountNavBean.displayView
                }"
                target="SecureShareCDM"/>
    </li>

</h:panelGroup>

The new flow is:

1. JSF reaches secureShareLinkDiv
        ↓
2. Before encoding, JSF asks:
   Is this component rendered?
        ↓
3. JSF evaluates the rendered expression
        ↓
4. isComponentEntitled("secureShareLinkDiv") executes
        ↓
5. SecurityPolicyHandler checks user entitlement
        ↓
6. Method returns true or false

If it returns false:

rendered = false
        ↓
JSF does not enter component encoding
        ↓
SecureShare is not rendered

If it returns true:

rendered = true
        ↓
JSF encodes secureShareLinkDiv
        ↓
JSF encodes secureShareLink
        ↓
SecureShare appears

The important difference is:

Old:
Start rendering
    → event changes rendered state

New:
Calculate rendered state
    → decide whether rendering should start

Side-by-side comparison

Point	Old JSF 2 flow	New Faces 4 flow

Access check location	f:event listener	rendered expression
Event type	preRenderComponent	No component event
Java method	checkComponentAccess(event)	isComponentEntitled(id)
Method result	void	boolean
Component obtained from	event.getComponent()	Component ID passed from XHTML
Visibility changed by	component.setRendered(false)	Boolean returned to JSF
Check timing	Component rendering has started	Before component encoding
Faces tree mutation	Yes	No
Mojarra implementation dependency	Higher	Lower
Faces 4 reliability	Timing-sensitive	Standard component behavior
Action listener	Unchanged	Unchanged
Action	Unchanged	Unchanged
Navigation target	Unchanged	Unchanged


Old entitlement check

Old code effectively did this:

public void checkComponentAccess(
        ComponentSystemEvent event) {

    UIComponent component =
            event.getComponent();

    String componentId =
            component.getClientId();

    boolean entitled =
            getSecurityPolicyHandler()
                    .isEntitlementInRole(componentId);

    if (!entitled) {
        component.setRendered(false);
    }
}

Notice the method performs two operations:

Ask security handler
        ↓
Mutate JSF component

The listener itself changes the component.

New entitlement check

Your existing boolean method does this:

public boolean isComponentEntitled(
        String componentId) {

    return getSecurityPolicyHandler()
            .isEntitlementInRole(componentId);
}

Now JSF performs the visibility decision:

Ask security handler
        ↓
Return true or false
        ↓
JSF decides whether to render

The Java method does not modify the component tree.

That is the cleaner design.

Initial page flow comparison

Old JSF 2

Browser requests page
        ↓
FacesServlet
        ↓
Restore/build component tree
        ↓
preRenderView
        ↓
checkRequestAccess(event)
        ↓
Render Response starts
        ↓
JSF reaches secureShareLinkDiv
        ↓
preRenderComponent
        ↓
checkComponentAccess(event)
        ↓
setRendered(false) if unauthorized
        ↓
Encode or skip component
        ↓
HTML response

Faces 4 with the UI-only change

Browser requests page
        ↓
FacesServlet
        ↓
Restore/build component tree
        ↓
preRenderView
        ↓
checkRequestAccess(event)
        ↓
Render Response starts
        ↓
JSF reaches secureShareLinkDiv
        ↓
Evaluate rendered expression
        ↓
isComponentEntitled("secureShareLinkDiv")
        ↓
Return true or false
        ↓
Encode or skip component
        ↓
HTML response

The page-level check is unchanged:

<f:event
        type="preRenderView"
        listener="#{
            applicationAccessBean.checkRequestAccess
        }"/>

Only the component-level access check changes.

Click flow comparison

Once SecureShare is visible, the click flow is essentially unchanged.

Old JSF 2 click flow

Click SecureShare
        ↓
h:commandLink submits form
        ↓
FacesServlet receives POST
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
Open response in SecureShareCDM

Faces 4 click flow

Click SecureShare
        ↓
h:commandLink submits form
        ↓
FacesServlet receives POST
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
Open response in SecureShareCDM

There is no meaningful difference here.

These remain the same:

<h:commandLink>
actionListener
action
target
disabled
styleClass

PrimeFaces 15 does not change this because you are using a standard Faces h:commandLink.

Postback difference

During the click postback, JSF may evaluate the parent’s rendered value while processing the component tree.

New flow:

Restore component tree
        ↓
Evaluate secureShareLinkDiv rendered
        ↓
isComponentEntitled(...) must still return true
        ↓
JSF can process the child secureShareLink
        ↓
handleClick and displayView execute

This is important:

> The entitlement must return the same value during the initial display and the click postback.



If the initial page says:

true

but the postback says:

false

then JSF may skip processing the link, and the action will not execute.

The entitlement information therefore needs to be stable for the request/session.

Why the new solution fixes your observed issue

You observed:

component.setRendered(false);

but later:

component.isRendered() == true

With the new code, no Java method calls:

setRendered(false)

Instead:

rendered="#{
    applicationAccessBean
        .isComponentEntitled('secureShareLinkDiv')
}"

The value is obtained directly whenever JSF asks whether the component should participate:

JSF asks isRendered()
        ↓
ValueExpression executes
        ↓
isComponentEntitled() returns false
        ↓
isRendered() is false

There is no separate event trying to overwrite the state.

One practical tradeoff

The old listener automatically obtained the component ID:

String componentId =
        event.getComponent().getClientId();

The new approach passes the ID explicitly:

isComponentEntitled('secureShareLinkDiv')

Therefore, the string must match componentSecurityPolicy.xml.

The tradeoff is:

Old:
Automatic client ID
but late component mutation

New:
Explicit security ID
but earlier reliable decision

Simple final answer

In JSF 2, your code allowed rendering to start and then used preRenderComponent to change the component to rendered=false. Mojarra 2 happened to respect that late change.

In Faces 4, changing the component from inside its own render event is timing-sensitive. The new flow calculates the entitlement through the rendered expression before JSF begins encoding the component.

The SecureShare click processing itself does not change:

actionListener
    ↓
action
    ↓
target window

Only the visibility decision changes:

Old: render event → mutate component
New: boolean expression → decide before encoding