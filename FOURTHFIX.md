This result isolates the problem: it also happens in a clean tab with the real account labels, without your application CSS, JSF, or PrimeFaces.

Therefore, stop changing midofficetools.css. Those changes cannot fix this reproduction.

What we know:

- Three short test labels render correctly.
- The real 26 account labels reproduce the distortion.
- The real labels contain very long text.

The leading suspect is a Chrome native-dropdown rendering problem triggered by the length/content of an option label. We can verify that without changing your application.

1. Test shorter labels in the copied tab only

Close the opened list with Esc.

Stay in the about:blank tab containing the copied account dropdown, open its Console, and run:

(() => {
    if (location.href !== 'about:blank') {
        console.error('Run this only in the copied about:blank tab.');
        return;
    }

    const select = document.querySelector('select');

    if (!select) {
        console.error('Copied dropdown not found.');
        return;
    }

    const options = Array.from(select.options);
    const originalLengths = options.map(option => option.text.length);

    options.forEach(option => {
        const originalText = option.text;
        const normalizedText = originalText.replace(/\s+/g, ' ').trim();

        option.title = originalText;
        option.textContent = normalizedText.length > 100
            ? normalizedText.slice(0, 100) + '…'
            : normalizedText;
    });

    select.style.width = '600px';
    select.style.maxWidth = '100%';

    console.log({
        optionCount: options.length,
        longestOriginalLabel: Math.max(0, ...originalLengths),
        testLabelLimit: 100
    });
})();

Now open that dropdown again.

This changes only the disposable copied dropdown. It does not modify your application, account values, or backend.

2. Interpret the result

Result	Conclusion

Shortened labels display correctly	Label length/content is a trigger; shortening display labels is a candidate workaround
Still distorted	Label shortening is insufficient; investigate option count and browser environment next


3. What the eventual application fix would preserve

If this test works, we can keep h:selectOneMenu and change only the display label:

Option value: unchanged account identifier
Option label: bounded, readable account description

We must retain enough information to distinguish accounts—particularly account number and account type. We should not blindly truncate production labels or change the selected values.

Please run this one test and tell me whether the shortened copied dropdown opens correctly. We now have a reproducible content-dependent issue, rather than an unconfirmed stylesheet problem.