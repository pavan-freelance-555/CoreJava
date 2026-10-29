Worked for 15s

I now understand the expected result correctly.

Your first screenshot shows the dropdown in its opened state, displaying a long native list over the page. The previous width: 500px CSS only changed the closed field and does not control how Chrome renders the opened native list.

The current XHTML logic is valid. The correct first step is to remove the unnecessary width, height, zoom and transform overrides.

Step 1: Use this exact h:selectOneMenu

For the first dropdown:

<c:if test="#{not accountsDropDownListBean.showBuyingPower}">

    <h:selectOneMenu
        id="accountDropdown"
        styleClass="native-account-dropdown"
        value="#{accountsDropDownListBean.selectedAccount}"
        disabled="#{disabled}"
        immediate="true"
        onchange="this.form.submit();"
        valueChangeListener="#{accountsDropDownListBean.accountOnChangeListener}">

        <f:selectItems
            value="#{accountsDropDownListBean.accountDropdownItems}" />

    </h:selectOneMenu>

</c:if>

For the buying-power section:

<c:if test="#{accountsDropDownListBean.showBuyingPower}">

    <h:panelGroup layout="block"
                  styleClass="block">

        <span class="data-title">
            #{tBundle.ACCOUNT}:
        </span>

        <h:panelGroup layout="block"
                      styleClass="data-pair">

            <h:selectOneMenu
                id="accountDropdownWithBuyingPower"
                styleClass="native-account-dropdown"
                value="#{accountsDropDownListBean.selectedAccount}"
                disabled="#{disabled}"
                immediate="true"
                onchange="this.form.submit();"
                valueChangeListener="#{accountsDropDownListBean.accountOnChangeListener}">

                <f:selectItems
                    value="#{accountsDropDownListBean.accountDropdownItems}" />

            </h:selectOneMenu>

        </h:panelGroup>

        <h:panelGroup layout="block"
                      styleClass="data-pair">

            <span class="data-title">
                #{tBundle.BUYING_POWER}:
            </span>

            <h:outputText
                value="#{accountsDropDownContextBean.currentAccount.buyingPower}">

                <f:convertNumber
                    type="currency"
                    locale="en_US" />

            </h:outputText>

        </h:panelGroup>

        <h:panelGroup layout="block"
                      styleClass="data-pair">

            <span class="data-title">
                #{tBundle.CASH_BALANCE}:
            </span>

            <h:outputText
                value="#{accountsDropDownContextBean.currentAccount.cashBalance}">

                <f:convertNumber
                    type="currency"
                    locale="en_US" />

            </h:outputText>

        </h:panelGroup>

        <div class="clearboth"></div>

    </h:panelGroup>

</c:if>

The important correction is:

onchange="this.form.submit();"

instead of:

onchange="return submit();"

This does not fix rendering, but it makes form submission unambiguous.

Step 2: Remove the previous CSS

Delete the previous rule containing:

width: 500px !important;
height: 24px !important;
max-width: 100% !important;
transform: none !important;
zoom: 1 !important;

Also remove:

select.account-dropdown-fix

The previous width rule cannot control Chrome’s opened native popup.

Step 3: Add only this minimal CSS

select.native-account-dropdown {
    -webkit-appearance: menulist !important;
    appearance: auto !important;
    font: inherit;
}

Do not style the <option> elements. Do not set a fixed height, transform or zoom.

This gives rendering responsibility back to Chrome, matching the behavior of the old browser as closely as possible.

Step 4: Reset Chrome zoom

Press:

Ctrl + 0

Confirm that Chrome shows:

Zoom: 100%

Then open the dropdown again.

Step 5: Check Windows display scaling

The known Chrome native-select problem is influenced by fractional display scaling.

Go to:

Windows Settings
→ System
→ Display
→ Scale

Temporarily test with:

100%

Sign out and sign back in if Windows requests it.

Then:

1. Start Chrome.
2. Press Ctrl + 0.
3. Open the application.
4. Click the dropdown.

If it works at Windows scale 100%, the problem is Chrome’s native popup calculation under display scaling—not JSF.

Step 6: Check the actual device scale

Open Developer Tools and run:

window.devicePixelRatio

Values such as these are generally predictable:

1
1.25
1.5
2

A value like this may trigger native-popup positioning/rendering problems:

0.953125
1.041666
1.333333333

Step 7: Test Chrome with device scaling forced to 1

Close every Chrome window.

Right-click the Chrome shortcut and select:

Properties

In the Target field, keep the existing Chrome path and add this after the closing quotation mark:

--force-device-scale-factor=1

Example:

"C:\Program Files\Google\Chrome\Application\chrome.exe" --force-device-scale-factor=1

Click:

Apply → OK

Start Chrome using that shortcut and test the dropdown.

This is an important diagnostic because Chromium’s native <select> issue has been associated with device scaling.

What application code can and cannot fix

h:selectOneMenu generates:

<select>
    <option>Account 1</option>
    <option>Account 2</option>
</select>

Your application controls:

- The closed field.
- Account values.
- Selected value.
- Disabled state.
- Form submission.
- Value-change listener.

Chrome/Windows controls:

- The opened popup.
- Popup height.
- Popup width limits.
- Popup location.
- Native scrolling.
- Native selected-row highlighting.
- Resolution and device scaling.

Therefore, no CSS change can guarantee that the opened h:selectOneMenu popup will look exactly the same in every Chrome version.

Final decision

If you must keep h:selectOneMenu, use the clean XHTML and minimal CSS above, then fix the Chrome/Windows scaling condition.

If the business requirement is:

> The opened list must look exactly the same in Chrome 155, 156 and future Chrome versions.



then a native h:selectOneMenu cannot guarantee that. A browser-independent HTML overlay component such as p:selectOneMenu would be required.

For now, the correct code change is only:

styleClass="native-account-dropdown"

with:

select.native-account-dropdown {
    -webkit-appearance: menulist !important;
    appearance: auto !important;
    font: inherit;
}

Do not keep the previous fixed-width CSS when testing the opened-list problem.