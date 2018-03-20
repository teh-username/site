---
title: JSX in a Nutshell
date: 2017-12-03 22:32:20
tags:
  - JSX
  - React
  - Virtual DOM
---


JSX is a thing that someone will surely mention when they discuss about React, making it look like JSX is a dependency of React. On the contrary, JSX and React are actually _independent_ of each other. 

You can actually use React without JSX like so: 

``` javascript
var hello = React.createElement(
  "h1",
  { id: "attr" },
  "Hello, world!",
  React.createElement(
    "p",
    null,
    "Hi!"
  ),
  "What's up?"
);
```

JSX helps enable some form of conversion so that React can understand what you mean with your fancy UIs. This post aims to discuss just that.

## JSX TL;DR

For the uninitiated ones, JSX stands for **J**ava**S**cript **X**ML (no, it actually doesn't, but it's nifty right?). That's all there is to it. JSX is just an extension to the [ECMAScript](https://stackoverflow.com/a/30113184) that allows you to write XML-like syntax natively.

The main reason though for using JSX is that it enables this chain (We'll be exploring each stage transition later):
{% raw %}
    <p style="text-align: center;">
        JSX -> Virtual DOM -> DOM
    </p>
{% endraw %}

If you're looking for the key take-away in this post, this should be it. JSX is basically just **Syntactic Sugar** in converting UI that we understand to the level that React understands (called "Virtual DOM"). Here's a visualization of that. 

Given a JSX element:

``` javascript
const element = (
    <h1 className="greeting">
        Hello, world!
    </h1>
)
```

It's equivalent conversion is:

``` javascript
const element = someFunction(
    "h1",
    { className: "greeting" },
    "Hello, world!"
);
```

where `someFunction` does the heavy lifting of converting it to Virtual DOM level. Note that the replacement from JSX tags to `someFunction` is handled by your transpiler.

## ELI5 Virtual DOM

"Virtual DOM" is simply the DOM you want to display in Javascript form. You may think of it as React's personal copy of the DOM (in a language only it can understand). We'll mention later why we need the DOM in Javascript form when we build our renderer.

Now, there is no standard way on how you'd do the conversion from JSX to Virtual DOM nor the structure of the object with regards to this. React has its own Virtual DOM structure that it deems optimal to make their library performant.

## JSX -> Virtual DOM

To explore the first part of the chain further, we'll create our own JSX -> Virtual DOM generator. Before that, few things first.

If you try using Babel's REPL, Codepen or JS Fiddle with JSX, the default function that it will call when it sees JSX is `React.createComponent` (Which is React's converter function). We'll need to override this somehow so that it uses our own function instead.

This is possible by explicitly setting the `pragma` at the top of your file as shown below:

``` javascript
/** @jsx toVDOM */

// JSX goes here

```

This'll tell the transpiler to use the function `toVDOM` instead. With that, we can now implement our very own converter as follows:

``` javascript
function toVDOM(node, attributes, ...rest) {
  const children = rest.length ? rest : null
  return {
    node,
    attributes,
    children
  };
}
```

So given a JSX element:

``` jsx
/** @jsx toVDOM */

const hello = (
  <h1 id="attr">
    Hello, world!
    <p>Hi!</p>
    What's up?
  </h1>
);
```

Our transpiler will convert it to:

``` javascript
const hello = toVDOM(
  "h1",
  { id: "attr" },
  "Hello, world!",
  toVDOM(
    "p",
    null,
    "Hi!"
  ),
  "What's up?"
);
```

Where our converter will generate the following Virtual DOM:

``` javascript
{
  "node": "h1",
  "attributes": {
    "id": "attr"
  },
  "children": [
    "Hello, world!",
    {
      "node": "p",
      "attributes": null,
      "children": [
        "Hi!"
      ]
    },
    "What's up?"
  ]
}
```

Just for comparison, here's the Virtual DOM for the same element if React would've had its way (Using `React.createComponent`):

``` javascript
{
  "type": "h1",
  "key": null,
  "ref": null,
  "props": {
    "id": "attr",
    "children": [
      "Hello, world!",
      {
        "type": "p",
        "key": null,
        "ref": null,
        "props": {
          "children": "Hi!"
        },
        "_owner": null,
        "_store": {}
      },
      "What's up?"
    ]
  },
  "_owner": null,
  "_store": {}
}
```

Notice that it kinda looks like the DOM but not really. I guess that's where the "Virtual" part came from. All that's left now is to convert this object into the DOM where our users can see all our hardwork.

## Virtual DOM -> DOM

We can code our DOM renderer as follows:

``` javascript
function render(virtualNode) {
  if (typeof virtualNode === 'string') {
    return document.createTextNode(virtualNode);
  }
  
  const element = document.createElement(virtualNode.node);
  
  Object.keys(virtualNode.attributes || {}).forEach(
    (attr) => element.setAttribute(attr, virtualNode.attributes[attr])
  );
  
  (virtualNode.children || []).forEach(
    (child) => element.appendChild(render(child))
  );
  
  return element;
}
```

It is important to note that our `render` function is tailor-made for our Virtual DOM structure. This is also the case for React and the others.

The renderer (and to some degree, the Virtual DOM converter as well) is one major point of competition for React and the other React-like libraries out there. This is where they try to paint your UI to the DOM as performant as possible. 

This is the exact reason why we need to bring the DOM into Javascript land. It is so that we can use our voodoo black magic computations to get the minimal DOM calls required to render our UI to our users.

Note that the Virtual DOM structure influences the renderer so every library that uses JSX has their own secret sauce (well, it's technically not a secret since they're open source but you get my point).

All that's left is to actually append it to the DOM:

``` javascript
const dom = render(hello);
document.body.appendChild(dom);
```

## JSX -> Virtual DOM -> DOM

Putting it all together gives us:

{% jsfiddle sy29cnwm js,result dark %}

## Edits

1. This entry was presented at a [local Javascript meetup](https://www.meetup.com/ManilaJavaScript/events/242890430/). Presentation slides can be acquired [here](jsx_in_a_nutshell.pdf).
