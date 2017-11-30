# Deprecation of HTML Imports styling the main document

This repository contains examples for mitigating deprecation of style application from HTML Imports [crbug/523952](https://bugs.chromium.org/p/chromium/issues/detail?id=523952).

## Examples

- example 1 ([code](https://github.com/TakayoshiKochi/deprecate-style-in-html-imports/tree/master/examples/ex1), [live demo](https://takayoshikochi.github.io/deprecate-style-in-html-imports/examples/ex1/master.html))

If a style that applies to the master document is in an import,
```html
<!DOCTYPE html>
<style>
.green {
  border : 2px solid green;
}
</style>
```

One obvious way to fix this is just move the stylesheet content to the master document manually.

Here's a script snippet that should run inside the same import, which will hoist the stylesheet to the master document, in case the master document cannot be modified.

```html
<script>
var importDoc = document.currentScript.ownerDocument;
var style = importDoc.querySelector('style');
document.head.appendChild(style);
</script>
```

See [fixed code in action](https://takayoshikochi.github.io/deprecate-style-in-html-imports/examples/ex1/master.html).

- example 2 ([code](https://github.com/TakayoshiKochi/deprecate-style-in-html-imports/tree/master/examples/ex2), [live demo](https://takayoshikochi.github.io/deprecate-style-in-html-imports/examples/ex2/master.html))

If `<link rel=stylesheet href=...>` is used in HTML Imports, the same strategy as example 1 can be used.

See [fixed code in action](https://takayoshikochi.github.io/deprecate-style-in-html-imports/examples/ex2/master.html).

- example 3 ([code](https://github.com/TakayoshiKochi/deprecate-style-in-html-imports/tree/master/examples/ex3), [live demo](https://takayoshikochi.github.io/deprecate-style-in-html-imports/examples/ex3/master.html))

- feature detection (by coutesy of PhistucK)

```
<link rel=import
      href="data:text/html,<style>body{--style-imported-detection:1;}</style>"
	  onload="console.log('<style> moved?', getComputedStyle(document.body).getPropertyValue('--style-imported-detection') === '1')"/>
```

#### Snippet to find problematic styles on given page

To find all problematic imports and stylesheets you can use following snippet.
```js
function findStylesheetsInImportedDocs(doc, map) {
    map = map || {};
    // map = map || new Map();
    Array.prototype.forEach.call(doc.querySelectorAll('link[rel=import]'), (e) => {
        const importedDoc = e.import;
        if (importedDoc && !map[importedDoc]) {
            const styles = importedDoc.querySelectorAll('style,link[rel=stylesheet]');
            if (styles.length) {
                map[importedDoc.URL] = {
                    doc: importedDoc,
                    styles: styles
                };
                // map.set(importedDoc, styles);
            }
            findStylesheetsInImportedDocs(importedDoc, map)
        }
    });
    return map;
}
console.log(findStylesheetsInImportedDocs(document));
```
available as a [gist](javascript:(function(){var%20screl=document.createElement('script');screl.setAttribute('type','text/javascript');screl.setAttribute('src','https://gist.githubusercontent.com/tomalec/47f0acd910a729d0b6a2e55061e5c26e/raw/4843f8a32d971f0465871a2d26cf6cc5c88e1229/findStylesheetsInAllHTMLImports.js');document.body.appendChild(screl);})();) or as a "<a href="javascript:(function(){var%20screl=document.createElement('script');screl.setAttribute('type','text/javascript');screl.setAttribute('src','https://gist.githubusercontent.com/tomalec/47f0acd910a729d0b6a2e55061e5c26e/raw/4843f8a32d971f0465871a2d26cf6cc5c88e1229/findStylesheetsInAllHTMLImports.js');document.body.appendChild(screl);})();">Find styles in HTML Imports bookmarklet</a>"


## Intent to Deprecate
[original mail with discussion](https://groups.google.com/a/chromium.org/d/topic/blink-dev/VZraFwqnp9Y/discussion)
### Primary eng email
kochi@chromium.org

### Summary
Currently stylesheets in HTML Imports (via either inline `<style>` or external `<link rel="stylesheet">`) is applied to their master document. Deprecate and eventually remove this behavior.
Eventually we can remove the [chapter 9 of HTML imports spec](http://w3c.github.io/webcomponents/spec/imports/#style-imports).
Current plan is to start showing deprecation message in 61 (Sep. 2017) and remove in 65 (Mar. 2018).

Note: stylesheets defined within `<template>` element, which is often used for stamping Shadow DOM contents, are not affected, as they are never applied to anything until they are stamped.

### Spec
The whole section of style in HTML Imports spec will be removed:

http://w3c.github.io/webcomponents/spec/imports/#style-imports

> The contents of the style elements and the external resources of the link elements in import
> must be considered as input sources of the style processing model [CSS2] of the master
> document.

A reference PR for removing this section:

https://github.com/w3c/webcomponents/pull/642

### Motivation
Applying style from stylesheets in HTML Imports (via `<script>` or `<link rel="stylesheet">`) has made Blink's style engine complex, and been confusing to web developers as they are unexpectedly applied to the master document, while none of the DOM tree inside HTML Imports is rendered.

This was originally introduced for giving "default style" for something in modular way, such as theming using HTML imports. But as `/deep/` and `::shadow` combinators of Shadow DOM v0 are deprecated and they are not even available in Shadow DOM v1, the value of this feature has become smaller.

Removing this will reduce the complexity which historically caused difficulty in making changes in style system (e.g. see [crbug/567021](http://crbug.com/567021) or [crbug/717506](http://crbug.com/717506)).

We will eventually be deprecating HTML imports as a whole once a concrete successor gets consensus; Deprecating the style application part is a stepping stone for it.

### Interoperability and Compatibility Risk
HTML Imports has been available only for Blink, and as of this writing (May 29, 2017) the stylesheet usage in imports is 0.044% ([source](https://www.chromestatus.com/metrics/feature/timeline/popularity/940)) within the whole HTML Imports usage (0.411%, [source](https://www.chromestatus.com/metrics/feature/timeline/popularity/455)), so currently 10+% of HTML import contains one or more stylesheets, although we do not have concrete data about how much of them are actually styling the master document.

As no other browsers than Chrome and Opera shipped HTML Imports, compatibility risk is only within Blink implementation and its users.

### Alternative implementation suggestion for web developers
Working around the issue is as easy as hoisting the stylesheet declaration to the master document manually or by script. For example, in an import, it is as simple as putting the following script in an import:

```js
<script>
var importDoc = document.currentScript.ownerDocument;
var style = importDoc.querySelector('style'); //  assuming only one <style>
document.head.appendChild(style);
</script>
```

See https://github.com/TakayoshiKochi/deprecate-style-in-html-imports for more
concrete examples.

For Polymer (both 1.x and 2.x) users, custom-style usage is affected by this.
In the latest versions of Polymer 1.x and 2.x, the custom-style element has 
been updated to move its styles into the main document.

1.x users should update to Polymer 1.10.1 or newer, and 2.x users should update to 
2.1.1 or newer to take advantage of the fix.
See https://github.com/Polymer/polymer/issues/4679 for the details.

### Usage information from UseCounter
https://www.chromestatus.com/metrics/feature/timeline/popularity/940

### OWP launch tracking bug
http://crbug.com/523952
and file a new bug for OWP launch tracking for removal.

### Entry on the feature dashboard
https://www.chromestatus.com/features/5144752345317376

### Requesting approval to remove too?
No at this point. The deprecation message, starting in 61, will include a target removal date of 65 (March 2018).
At that time I plan to send a separate Intent to Remove where we can reevaluate the effect of the deprecation message on driving down usage.
