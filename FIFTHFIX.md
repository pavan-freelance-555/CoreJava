Worked for 15s

We now have a working workaround: normalizing and shortening the option labels, together with the 600px field width, fixes the copied dropdown.

This does not prove that 100 characters is a Chrome limit—it is simply the length that passed your test. We can apply the same workaround while keeping h:selectOneMenu and preserving the submitted account identifiers.

1. Add this JavaScript function

Place this block in AccountsDropDownList.xhtml, immediately inside <ui:composition>, before the dropdowns:

<script type="text/javascript">
    //<![CDATA[
    window.prepareAccountDropdown = function (select) {
        if (!select) {
            return;
        }

        Array.from(select.options).forEach(function (option) {
            // Do not shorten an already-processed label again.
            if (option.hasAttribute('data-original-account-label')) {
                return;
            }

            const originalLabel = option.label;
            const originalValue = option.value;

            const normalizedLabel =
                originalLabel.replace(/\s+/g, ' ').trim();

            const displayLabel = normalizedLabel.length > 100
                ? normalizedLabel.slice(0, 100) + '\u2026'
                : normalizedLabel;

            /*
             * Preserve the submitted account identifier before
             * modifying any displayed text.
             */
            option.value = originalValue;

            option.setAttribute(
                'data-original-account-label',
                originalLabel
            );

            option.title = originalLabel;
            option.textContent = displayLabel;
            option.label = displayLabel;
        });

        const selectedOption = select.options[select.selectedIndex];

        if (selectedOption) {
            select.title =
                selectedOption.getAttribute('data-original-account-label')
                || selectedOption.label;
        }
    };
    //]]>
</script>

Preserving option.value is important: if an option has no explicit value attribute, its submitted value otherwise comes from its text. 

2. Update both dropdowns

For the first dropdown, use:

<h:selectOneMenu
    id="accountDropdown"
    styleClass="native-account-dropdown"
    onfocus="prepareAccountDropdown(this);"
    onmousedown="prepareAccountDropdown(this);"
    onchange="this.form.submit();"
    immediate="true"
    value="#{accountsDropDownListBean.selectedAccount}"
    disabled="#{disabled}"
    valueChangeListener="#{accountsDropDownListBean.accountOnChangeListener}">

    <f:selectItems
        value="#{accountsDropDownListBean.accountDropdownItems}" />

</h:selectOneMenu>

Apply the same attributes to the second dropdown, keeping its different ID:

id="accountDropdownWithBuyingPower"

The function runs before opening through mouse interaction or keyboard focus. It also processes newly rendered dropdowns when they receive focus.

Do not change the bean, f:selectItems, listener, immediate, or existing conditional sections.

3. Replace the earlier experimental CSS

Keep this scoped rule:

select.native-account-dropdown {
    width: 600px;
    max-width: 100%;
}

Remove only the experimental dropdown overrides previously added for fixed heights, zoom, transforms, and parent borders. Leave unrelated application CSS untouched.

4. Verify on the actual application

After deployment and a hard reload:

1. Open the Account dropdown and confirm all 26 options display.
2. Select a different account.
3. Confirm accountOnChangeListener receives the correct account identifier.
4. Confirm the page shows the selected account’s details.
5. Repeat using keyboard navigation.

Before release, confirm that account number and account type remain visible and distinguish every option. Full labels are retained in title, but native option tooltips are not consistently displayed; if users must see every account-holder name, provide that full description separately rather than relying on the tooltip.

This is a display-only workaround. The actual application still needs the above selection test before we call the defect resolved.