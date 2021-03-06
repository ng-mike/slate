
<br/>
<p align="center"><strong>Previous:</strong><br/><a href="./saving-to-a-database.md">Saving to a Database</a></p>
<br/>

# Saving and Loading HTML Content

In the previous guide, we looked at how to serialize the Slate editor's content and save it for later. But we only covered the [`Plain`](../reference/serializers/plain.md) and [`Raw`](../reference/serializers/raw.md) serialization techniques.

What if you want to save the content as HTML? It's a slightly more involved process, but this guide will show you how to do it.

Let's start with a basic editor:

```js
import { Editor } from 'slate'

class App extends React.Component {

  constructor(props) {
    super(props)
    this.state = {
      state: Plain.deserialize('')
    }
  }

  render() {
    return (
      <Editor
        state={this.state.state}
        onChange={state => this.onChange(state)}
      />
    )
  }

  onChange(state) {
    this.setState({ state })
  }

}
```

That will render a basic Slate editor on your page.

Now... we need to add the [`Html`](../reference/serializers/html.md) serializer. And to do that, we need to tell it a bit about the schema we plan on using. For this example, we'll work with a schema that has a few different parts:

- A `paragraph` block.
- A `code` block for code samples.
- A `quote` block for quotes...
- And `bold`, `italic` and `underline` formatting.

By default, the `Html` serializer, knows nothing about our schema just like Slate itself. To fix this, we need to pass it a set of `rules`. Each rule defines how to serialize and deserialize a Slate object.

To start, let's create a new rule with a `deserialize` function for paragraph blocks.

```js
const rules = [
  {
    deserialize(el, next) {
      if (el.tagName == 'p') {
        return {
          kind: 'block',
          type: 'paragraph',
          nodes: next(el.children)
        }
      }
    }
  }
]
```

If you've worked with the [`Raw`](../reference/serializers/raw.md) serializer before, the return value of the `deserialize` should look familiar! It's just the same raw JSON format.

The `el` argument that the `deserialize` function receives is just a [`cheerio`](https://github.com/cheeriojs/cheerio) element object. And the `next` argument is a function that will deserialize any `cheerio` element(s) we pass it, which is how you recurse through each nodes children.

Okay, that's `deserialize`, now let's define the `serialize` property of the paragraph rule as well:

```js
const rules = [
  {
    deserialize(el, next) {
      if (el.tagName == 'p') {
        return {
          kind: 'block',
          type: 'paragraph',
          nodes: next(el.children)
        }
      }
    },
    serialize(object, children) {
      if (obj.kind == 'block' && obj.type == 'paragraph') {
        return <p>{children}</p>
      }
    }
  }
]
```

The `serialize` function should also feel familiar. It's just taking [Slate models](../reference/models) and turning them into React elements, which will then be rendered to an HTML string.

The `object` argument of the `serialize` function will either be a [`Node`](../reference/models/node.md), a [`Mark`](../reference/models/mark.md) or a special immutable [`String`](../reference/serializers/html.md#ruleserialize) object. And the `children` argument is a React element describing the nested children of the object in question, for recursing.

Okay, so now our serializer can handle `paragraph` nodes.

Let's add the other types of blocks we want:

```js
const BLOCK_TAGS = {
  p: 'paragraph',
  blockquote: 'quote',
  pre: 'code'
}

const rules = [
  {
    // Switch deserialize to handle more blocks...
    deserialize(el, next) {
      const type = BLOCK_TAGS[el.tagName]
      if (!type) return
      return {
        kind: 'block',
        type: type,
        nodes: next(el.children)
      }
    },
    // Switch serialize to handle more blocks...
    serialize(object, children) {
      if (obj.kind != 'block') return
      switch (obj.type) {
        case 'paragraph': return <p>{children}</p>
        case 'quote': return <blockquote>{children}</blockquote>
        case 'code': return <pre><code>{children}</code></pre>
      }
    }
  }
]
```

Now each of our block types is handled.

You'll notice that even though code blocks are nested in a `<pre>` and a `<code>` element, we don't need to specifically handle that case in our `deserialize` function, because the `Html` serializer will automatically recurse through `el.children` if no matching deserializer is found. This way, unknown tags will just be skipped over in the tree, instead of their contents omitted completely.

Okay. So now our serializer can handle blocks, but we need to add our marks to it as well. Let's do that with a new rule...


```js
const BLOCK_TAGS = {
  blockquote: 'quote',
  p: 'paragraph',
  pre: 'code'
}

const MARK_TAGS = {
  em: 'italic',
  strong: 'bold',
  u: 'underline',
}

const rules = [
  {
    deserialize(el, next) {
      const type = BLOCK_TAGS[el.tagName]
      if (!type) return
      return {
        kind: 'block',
        type: type,
        nodes: next(el.children)
      }
    },
    serialize(object, children) {
      if (obj.kind != 'block') return
      switch (obj.type) {
        case 'code': return <pre><code>{children}</code></pre>
        case 'paragraph': return <p>{children}</p>
        case 'quote': return <blockquote>{children}</blockquote>
      }
    }
  },
  // Add a new rule that handles marks...
  {
    deserialize(el, next) {
      const type = MARK_TAGS[el.tagName]
      if (!type) return
      return {
        kind: 'mark',
        type: type,
        nodes: next(el.children)
      }
    },
    serialize(object, children) {
      if (obj.kind != 'mark') return
      switch (obj.type) {
        case 'bold': return <strong>{children}</strong>
        case 'italic': return <em>{children}</em>
        case 'underline': return <u>{children}</u>
      }
    }
  }
]
```

Great, that's all of the rules we need! Now let's create a new `Html` serializer and pass in those rules:

```js
import { Html } from 'slate'

const html = new Html({ rules })
```

And finally, now that we have our serializer initialized, we can update our app to use it to save and load content, like so:



```js
const initialState = (
  localStorage.get('content') ||
  '<p></p>'
)

class App extends React.Component {

  constructor(props) {
    super(props)
    this.state = {
      state: html.deserialize(initialState)
    }
  }

  render() {
    return (
      <Editor
        state={this.state.state}
        onChange={state => this.onChange(state)}
        onDocumentChange={(document, state) => this.onDocumentChange(document, state)}
      />
    )
  }

  onChange(state) {
    this.setState({ state })
  }

  onDocumentChange(document, state) {
    const string = html.serialize(state)
    localStorage.set('content', string)
  }

}
```

And that's it! When you make any changes in your editor, you should see the updated HTML being saved to Local Storage. And when you refresh the page, those changes should be carried over.
