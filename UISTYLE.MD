The scoped width CSS should work across Chrome versions, including 155 and 156, because it uses standard CSS. However, there is one important limitation:

h:selectOneMenu renders a native HTML <select>. Your CSS can reliably control the closed field’s width and height, but Chrome and the operating system control the appearance of the opened option popup.

Use this minimal cross-version solution:

<h:selectOneMenu
    styleClass="account-dropdown-fix"
    onchange="return submit();"
    immediate="true"
    value="#{accountsDropDownListBean.selectedAccount}"
    disabled="#{disabled}"
    valueChangeListener="#{accountsDropDownListBean.accountOnChangeListener}">

    <f:selectItems
        value="#{accountsDropDownListBean.accountDropdownItems}" />

</h:selectOneMenu>

select.account-dropdown-fix {
    width: 500px !important;
    max-width: 100% !important;
    height: 24px !important;
    box-sizing: border-box !important;

    -webkit-appearance: auto !important;
    appearance: auto !important;

    transform: none !important;
    zoom: 1 !important;
}

This should work in:

- Chrome 155
- Chrome 156
- Future Chrome versions
- Microsoft Edge
- Firefox

It is safer because:

- The rule applies only to this account dropdown.
- It does not detect or depend on a specific browser version.
- It does not use Chrome-version-specific JavaScript.
- appearance:auto allows each browser to use its supported native dropdown.
- Explicit width prevents Chrome from calculating the control as the full parent width.

Do not add browser-version conditions such as:

if (chromeVersion === 156) {
    // Change width
}

That would be fragile.

One correction: if your existing height was previously correct, start with only:

select.account-dropdown-fix {
    width: 500px !important;
    max-width: 100% !important;
    box-sizing: border-box !important;
}

Only add height, appearance, transform, and zoom if the control is still visually distorted. This minimizes the chance of changing the older Chrome appearance.

After deployment, test at least:

Browser	Required check

Chrome 155	Field width and selection
Chrome 156	Field width and opened option list
Edge	Field width and selection
Browser zoom 100%	Baseline
Browser zoom 125%	Responsive behavior


So, yes—the explicit scoped-width fix is cross-version. But no application CSS can guarantee that the native opened popup looks pixel-for-pixel identical in every Chrome version, because Chrome itself renders that part.