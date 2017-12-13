---
layout: post
title:  "2016 Presidential Erection"
date:   2017-12-01T19:00:00-08:00
description: "Chrome extension to replace all 'elections' with 'erections'"
categories:
---

If you're like me, the 2016 American Presidential election probably bummed you out. I made a Chrome extension to drastically improve my browsing experience.

#### The Goal

Create a chrome extension the replaces all "Election"s with "Erection"s.

See the code [here](https://github.com/pjskennedy/2016-presidential-erection).
See the extension for download [here](https://chrome.google.com/webstore/detail/2016-erection/pgnkdnfoandiffjncfedalpjflldhfed).

<img src="/assets/images/presidential-election/screenshot.png" style="max-width: 100%;">

This was pretty easy, Google and a little bit of Javascript makes this a pretty painless project.

First we need a Javascript function that replaces a dom element with the appropriate _erected_ text.

```js
// Takes in a dom element "node", a regex to match "elections", and a replacement text of "erections"
var replaceText = function(node, regex, erection) {
  var text = node.nodeValue;
  var replacedText = text.replace(regex, erection);
  if (replacedText !== text) {
    element.replaceChild(document.createTextNode(replacedText), node);
  }
}

var erectrify = function(node) {
  replaceText(node, /\belection/g, "erection");
  replaceText(node, /\bElection/g, "Erection");
  replaceText(node, /\bELECTION/g, "ERECTION");
}
```

Now we need some quick javascript to find only the text bodies. We can do this by querying the elements in the dom of `nodeType` 3. This node type is of type `Text` and "Represents textual content in an element or attribute" (read more [here](https://www.w3schools.com/jsref/prop_node_nodetype.asp)).

```js
var elements = document.getElementsByTagName('*');

for (var i = 0; i < elements.length; i++) {
  var element = elements[i];
  for (var j = 0; j < element.childNodes.length; j++) {
    var node = element.childNodes[j];
    // nodeType of 3 is "Node.TEXT_NODE"
    if (node.nodeType === 3) {
      erectrify(element.childNodes[j]);
    }
  }
}
```

Now add a manifest to run this script in the browser on every page and we're off to the races:

```js
{
  "manifest_version": 2,
  "name": "2016 Erection",
  "description": "Changes all uses of 'election' with 'erection'.",
  "version": "1.4",
  "content_scripts": [
    {
      "matches": ["*://*/*"],
      "js": ["content.js"],
      "run_at": "document_end"
    }
  ]
}
```
