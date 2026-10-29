This test works correctly. Chrome can render a basic native dropdown normally on your machine. So we should not keep changing graphics acceleration or Windows settings.

However, your test has only three short labels; the application has 26 options with long labels. We must now test those same options without application CSS.

Next step: copy the actual dropdown into a clean tab

Return to the Mid Office Tools tab, open Console, and run:

(() => {
    const original = document.getElementById('accountDropdown');

    if (!original) {
        console.error('Cannot find accountDropdown.');
        return;
    }

    const testWindow = window.open('', '_blank');

    if (!testWindow) {
        console.error('Popup blocked. Allow the popup and retry.');
        return;
    }

    const cleanSelect = testWindow.document.createElement('select');

    for (const originalOption of original.options) {
        const option = testWindow.document.createElement('option');

        option.textContent = originalOption.text;
        option.selected = originalOption.selected;
        option.disabled = originalOption.disabled;

        cleanSelect.appendChild(option);
    }

    testWindow.document.body.appendChild(cleanSelect);
})();

This copies the displayed account labels into a local tab. It does not call a service, submit a form, or modify your account data.

Open that new dropdown and check:

Result	Meaning

All 26 account options display correctly	Application styling or surrounding page context triggers the issue
The same oversized blank popup appears	The issue depends on the real option content/count, potentially triggering a browser rendering bug


Please send the result of this exact test. It preserves the real labels and option count while removing application CSS—the key difference the three-item test did not cover. Keep your XHTML and Java unchanged until we have that result.