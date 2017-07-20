---
layout: post
title: "Becoming a Draft.js Convert"
date: 2017-07-20 14:21
author: eve
categories: [javascript]
---

Artsy's [Writer](https://github.com/artsy/positron) publishing tool, used primarily by our [Editorial Team](https://artsy.net/magazine), is a custom CMS for creating dynamic magazine-style content. Rich text editing is a crucial element, and using content editable divs for the task was becoming unruly— especially for managing application states as React become more deeply ingrained throughout the repository. We recently decided to retire the use of content editable entirely, previously implemented throughout the app via the Guardian's [Scribe library](https://github.com/guardian/scribe), in favor of Facebooks's React-based [Draft.js](https://draftjs.org) rich text editor.

In this post we will take a look at some of the basics using Draft.js, as well as implementing a customized entity, and saving text as exported HTML.

<!-- more -->

### What is Draft.js?
Draft.js takes a granular approach to storing rich text, eschewing HTML entirely for an immutable JSON object. Known as the `EditorState`, a developer always has access to a character-by-character representation of the editor component's state. Draft.js provides the `Editor` component, which is highly customizable, but only a few rich text features are ready to go out of the box. Spell check is a simple boolean prop, and several event hooks are available for mouse, keyboard and clipboard events. All it takes to get a basic editor running is an `onChange` function, and an `onClick` event to focus. Note that the click event must be applied to the parent container, rather than the editor itself.

```javascript
// A functional text editor with spell-check
import React, { Component } from 'react'
import ReactDOM from 'react-dom'
import { Editor, EditorState } from 'draft-js'

class RichText extends Component {
  constructor(props) {
    super(props)
    this.state = {
      editorState: EditorState.createEmpty()
    }
    this.focus = () => this.refs.editor.focus()
    this.onChange = this.onChange.bind(this)
  }

  onChange(editorState) {
    this.setState({ editorState })
  }

  render() {
    return (
      <div onClick={this.focus}>
        <Editor
          ref='editor'
          editorState={this.state.editorState}
          onChange={this.onChange}
          spellCheck={true} />
      </div>
    )
  }
}

export default RichText
```

Surprisingly, HTML support in the library is minimal. Though DraftJS can import HTML elements into an empty editor component, Facebook makes no assumptions about what format your text will be saved in. An [external library](https://github.com/nikgraf/awesome-draft-js#common-utilities) is recommended if you plan to save your rich text as HTML. Artsy uses data attributes and selectors from our articles' html to trigger JavaScript on the client side, so we chose to use the [draft-convert](https://github.com/HubSpot/draft-convert) library for importing and exporting our HTML data on initialization and save.


## Case Study: Artist Follow Button
An excellent example of a custom feature we've built for DraftJS is the [artist follow button](https://github.com/artsy/force/tree/master/desktop/components/follow_button). This component is used both in articles and throughout Artsy.net:

![Artist Follow Button](/images/2017-07-20-converting-draft/follow-button.gif)

When linking to an artist's page in an article, our editorial team can optionally include this widget, which prompts users to follow the linked artist on their Artsy profile. We create a hook for this feature by adding an ‘is-follow-link’ class to the artist link, and appending it with a second, empty a tag that includes a 'data-id' attribute referencing our artist. Below is an example of this text as it is saved and rendered on the front-end.

```html
<h2>
  <a href="https://www.artsy.net/artist/ej-hill" class="is-follow-link">EJ Hill</a>
  <a data-id="ej-hill" class="entity-follow artist-follow"></a>
</h2>
```

### Implementing decorators

To get this running in DraftJS we will first need to wire up our editor to support links. This is accomplished using decorator patterns. Draft provides you a `CompositeDecorator`, which accepts one argument-- an array of decorators for each custom entity you want to include. A draft decorator combines two functions: a strategy for finding an entity, as well as a component that renders your entity in the desired format.

The `CompositeDecorator` must be passed to the `editorState` before it mounts. If you are importing existing data to the editor, you will need to set the `editorState`, again with the decorator, after the component mounts. Our implementation is adapted from Facebook's link example, which you can check out [here](https://github.com/facebook/draft-js/tree/master/examples/draft-0-10-0/link).

```javascript
import React, { Component } from 'react'
import ReactDOM from 'react-dom'
import { Editor, EditorState, CompositeDecorator } from 'draft-js'
import { findLinkEntities, Link } from './decorators'

const decorator = new CompositeDecorator([
  {
    strategy: findLinkEntities,
    component: Link
  }
])

class RichText extends Component {
  constructor(props) {
    super(props)
    this.state = {
      editorState: EditorState.createEmpty(decorator)
    }
    this.focus = () => this.refs.editor.focus()
    this.onChange = this.onChange.bind(this)
  }

  ...

```

### OnChange - maybe move this or give example of mutating editor
Because drafts editorState is immutable, every change requires duplicating the editorState, rather than making changes to it directly. Our onchange event will simply replace our current editorState with a new copy with changes applied to it.

### Decorators up close

Now take a look at the functions our `CompositeDecorator` requires. We use our decorator to search for [entities](https://draftjs.org/docs/api-reference-entity.html)-- these are used to find and render blocks that accept additional props in addition to text, such as a link or image.

The `findLinkEntity` function cycles through all the blocks in our editor state, and then checks the metadata of each character looking for instances of a Link. The object has props that contain our href/URL, but may also include class names, name, and data attributes.

```javascript
import React from 'react';
import { ContentState, Editor, EditorState } from 'draft-js'

const findLinkEntities = (contentBlock, callback, contentState) => {
  contentBlock.findEntityRanges(
    (character) => {
      const entityKey = character.getEntity();
      return (
        entityKey !== null &&
        contentState.getEntity(entityKey).getType() === 'LINK'
      );
    },
    callback
  );
}

const Link = (props) => {
  const {url} = props.contentState.getEntity(props.entityKey).getData();
  return (
    <a href={url}>
      {props.children}
    </a>
  );
};

module.exports = {
  Link: Link,
  findLinkEntities: findLinkEntities
}
```

Our `CompositeDecorator` passes `findLinkEntity` to the `link` component as a callback. After a link entity is matched, the CompositeDecorator returns the formatted link. In the `link` component we can render our entity's data in whatever format we choose. Because Artsy uses both standard links as well as our artist follow links, we use a conditional to look for the is-follow-button prop, and render in formats for each.

### Creating New Link Entities
Now we have the ability to find and render existing links, but we can't yet create one in our editor. For this we will need to add an additional input field for a URL, a button to display our field conditionally, and a submit button to accept the user input.

- input component

To accept input from this form, we will pass an onSubmit function: confirmLink, which will convert our input into a draft link entity.

-  A closer look at confirmLink

### adding an artist flag to confirmLink
Our link input now accepts a url and can render it as a link in the editor. however, we haven't yet accounted for our Artist follow Button. We will need to add an argument to our promptForLink event alerting us to the link type, which we will save in our state. We also need to update our confirmLink to use the linkType

### converting existing link entities
We use the draft-convert library to export our editor state as HTML on every change, and save it to our components state. This library supports most rich-text elements out of the box, but not entities- including a standard link. Anything you use a decorator for, will also require a format to be declared for out conversion. Below is our convert function, which is called on each onChange event.

- code of convert to HTML 

One caveat of this library, and others including x and x is the use of the Entity.Create method- it will be deprecated in the next major version and throws a warning. 

### styling our draft editor


- link component

### creating links with metadata

Entities are nested within blocks.  Let's think about react, and how freestanding elements cannot exist side-by-side unless nested within a parent container.  In draft, our text works the same way. Consider a paragraph with a link inside-- in draft, text before or after the link would be wrapped in spans.  here it traverses a block by character seeking a place where the link starts and ends. 
