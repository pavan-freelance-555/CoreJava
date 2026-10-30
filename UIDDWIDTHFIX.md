Then width: 100% is not keeping the width stable in your layout. The reliable next change is to capture the dropdown’s current width before shortening its labels.

Do not replace the working label-shortening logic. Add these lines at the beginning of prepareAccountDropdown(), immediately after the null check and before Array.from(select.options)...:

// Preserve the existing field width before shortening labels.
if (!select.hasAttribute('data-account-width-locked')) {
    const currentWidth = select.getBoundingClientRect().width;

    if (currentWidth > 0) {
        select.style.setProperty('box-sizing', 'border-box', 'important');
        select.style.setProperty('width', currentWidth + 'px', 'important');
        select.style.setProperty('min-width', currentWidth + 'px', 'important');
        select.style.setProperty('max-width', currentWidth + 'px', 'important');

        select.setAttribute('data-account-width-locked', 'true');
    }
}

Your function should therefore start like this:

window.prepareAccountDropdown = function (select) {
    if (!select) {
        return;
    }

    // ADD THE WIDTH-LOCK BLOCK HERE.

    Array.from(select.options).forEach(function (option) {
        // Keep your existing working label-shortening code.
    });

    // Keep the remaining existing code.
};

Keep both existing event attributes:

onfocus="prepareAccountDropdown(this);"
onmousedown="prepareAccountDropdown(this);"

Remove the experimental width: 100% dropdown rule and any 600px assignments, then reload the page completely before testing. Otherwise, the script may capture the already-shrunken width.

The sequence becomes:

1. Dropdown initially appears at its original long width.
2. On focus/click, the script saves that width in pixels.
3. Labels are shortened.
4. The field keeps its saved width instead of shrinking.

Trade-off: the field width stays fixed until the component is rendered again or the page reloads; it will not automatically resize with the browser window.

This controls the closed field’s width. If only the floating opened list is narrower while the field itself stays unchanged, that is a separate native-popup sizing behavior—not a failure of this width lock.