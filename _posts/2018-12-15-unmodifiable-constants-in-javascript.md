---
layout: post
title: Unmodifiable Constants in JavaScript
date: 2018-12-15 10:43 -0500
---

TL;DR: [https://www.npmjs.com/package/enforce-constants](https://www.npmjs.com/package/enforce-constants) lets you make sure your objects raise an error if you access a key that has not been defined.

---

One of the more difficult aspects of programming in JavaScript is the mutability of variables. Don't get me wrong, coming from a Ruby background, there are some times when that can be exactly what you need in code. However, many times when writing JavaScript I find that I want to have a true "constant" that can't be redefined once created. Additionally, I often use configuration objects or reference objects to hold often-typed strings so that I can get better error messages in the event I make a typo.

This post will walk through creating a construct that allows us to do the following:

```javascript
const applicationType = {
    html: 'application/html',
    text: 'application/text',
    doc: 'application/msword',
    docx: 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
}

module.exports = enforceConstants(applicationType);

//
applicationType.html // 'application/html'
applicationType.pptx // Error: 'pptx' is not defined on applicationType
```

This construct can be applied anywhere you want to define a set of data that you don't want redefined later. Additionally, it can ensure that you _only_ access things that have been defined - no silent returning of `undefined` (which can make debugging a real pain if you don't spot it early... or so I've heard).

The bulk of the code is:

```javascript
'use strict';

function enforceConstants(obj, name) {
  const handler = {};
  handler.enforceConstantsName = name;

  handler.set = function(obj, prop) {
    throw new Error('Cannot modify an object enforcing constant access');
  }

  handler.deleteProperty = function(obj, prop) {
    throw new Error('Cannot delete properties of an object enforcing constant access');
  }

  handler.get = function(obj, prop) {
    if (prop in obj) return obj[prop];

    const name = handler.enforceConstantsName ? `[${handler.enforceConstantsName}]` : null;
    const errorString = `${prop} is not defined on an object enforcing constant access`;
    throw new Error([errorString, name].join(' ').trim());
  }

  const proxiedObject = new Proxy(Object.freeze(obj), handler);
  return proxiedObject;
}
```

If you're familiar with the JavaScript Proxy object, this is a very straightforward use of it. We take the object we are given and pass it in to a Proxy constructor, along with a custom handler. The `handler` is where we define our custom logic. In this case, we want to throw an error anytime a user attempts to write or delete a property, which is handled with `set` and `deleteProperty` functions, respectively. The `get` method is only slightly more complicated - if the property exists on the object, we return it. Otherwise, we raise an error and include the name of the object if it was provided at the beginning.

The NPM module has a few more additions in the `get` code that allow it to be passed successfully to `console.log`. Node uses some custom symbol definitions to know what to print (i.e., why a function is type `function`, object is type `Object object`, etc.). Because we are intercepting _all_ calls to the object, we have to manually handle those. However, it does give us the ability to customize what is printed to the console, hopefully providing develoeprs with a bit more context around the object, where it's being used, and its original intent.

Feel free to check out the code at [https://github.com/jacobsmith/enforce-constants](https://github.com/jacobsmith/enforce-constants) and offer any suggestions or issues there!