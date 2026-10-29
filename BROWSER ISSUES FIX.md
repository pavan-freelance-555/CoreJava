These screenshots confirm two important points:

1. Your CSS is being loaded and applied. DevTools shows select.native-account-dropdown with active appearance: auto and font: inherit.
2. The oversized white area and blue highlight belong to the opened native dropdown. It even covers DevTools. My earlier suggestion that this was a parent border was incorrect.

Do not change Java, the listener, or more layout CSS yet.

Next step: isolate option styling from Chrome rendering

Close the dropdown with Esc. In DevTools → Console, run this:

(() => {
    const select = document.getElementById('accountDropdown');

    if (!select) {
        console.error('Account dropdown not found.');
        return;
    }

    console.log({
        optionCount: select.options.length,
        size: select.size,
        multiple: select.multiple,
        devicePixelRatio: window.devicePixelRatio
    });

    console.table(
        [select, ...Array.from(select.options).slice(0, 3)].map(el => {
            const css = getComputedStyle(el);

            return {
                element: el.tagName,
                fontSize: css.fontSize,
                lineHeight: css.lineHeight,
                height: css.height,
                minHeight: css.minHeight,
                padding: css.padding,
                color: css.color,
                background: css.backgroundColor,
                display: css.display
            };
        })
    );
})();

This prints styling and counts—not account names. Send the output.

Also test a clean dropdown with dummy values

Run this separately:

(() => {
    const testWindow = window.open('', '_blank');

    if (!testWindow) {
        console.error('Popup blocked. Allow this diagnostic popup.');
        return;
    }

    const select = testWindow.document.createElement('select');

    ['Test Account A', 'Test Account B', 'Test Account C'].forEach(label => {
        const option = testWindow.document.createElement('option');
        option.textContent = label;
        select.appendChild(option);
    });

    testWindow.document.body.appendChild(select);
})();

Open the dropdown in that new tab.

Result	Next investigation

Dummy dropdown is also distorted	Browser/native-popup environment; application CSS is not required to reproduce it
Dummy dropdown works normally	Account option content, application styling, or page context


Please provide these two results before making another code change. We have now confirmed that the stylesheet is applied; the missing evidence is whether the distortion occurs in a completely unstyled native dropdown. That determines the appropriate fix.