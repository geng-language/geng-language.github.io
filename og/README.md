# The link preview image

`static/og.png` (1200×630) is what chat apps and social sites show for a
link to the site. It is rendered from `og.html` here, which uses the site's
fonts and pattern from `../static/`: open it in a browser at 1200×630 and
take a screenshot, or with Playwright:

```js
const p = await browser.newPage({ viewport: { width: 1200, height: 630 } });
await p.goto('file://' + path.resolve('og/og.html'));
await p.evaluate(() => document.fonts.ready);
await p.screenshot({ path: 'static/og.png' });
```

The checks-report excerpt in it is puritty's, from geng-lang's
`bench/proof-report` (PR13), as the home page's is.
