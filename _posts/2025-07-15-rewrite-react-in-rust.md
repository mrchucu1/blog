# Rusting the Gears of React: A "From-Scratch" Journey

**Posted by: Diego Navarro ([@mrchucu1](https://github.com/mrchucu1))**

As a Lead Software Engineer at Meta, I spend most of my days working within massive, highly-optimized codebases. But to stay sharp, it's critical to go back to first principles. I recently tasked myself and a senior engineer with a fascinating challenge: **What would it take to rewrite the core of React... in Rust?**

This isn't about creating the next production-ready UI framework. Projects like [Dioxus](https://dioxuslabs.com/) and [Yew](https://yew.rs/) are already doing incredible work in that space. Instead, this is a pedagogical exercise—a way to deeply understand the mechanics of a modern UI library by building it from the ground up, while leveraging the power and safety of Rust and WebAssembly.

This post documents the first leg of our journey: from a blank slate to rendering our first static UI in a browser, entirely from Rust.

## Session 1: The Core Atom - The "Element"

The most fundamental unit in React isn't the component; it's the **Element**. When you write `<div />`, you aren't creating a DOM node. You're calling `React.createElement()`, which returns a lightweight JavaScript object—a blueprint describing what *should* be rendered.

Our first task was to model this blueprint in Rust. A plain struct was the obvious choice.

```rust
#[derive(Debug, PartialEq)]
pub struct Element {
    pub tag_name: String,
    pub children: Vec<Element>,
}
```

This is as simple as it gets, but it's the core concept. It's a tree-like data structure where an element has a tag (like `"div"`) and can contain other elements.

## Session 2: Props, Text, and the Virtual DOM

Our initial struct was too simple. Elements have attributes (`id`, `class`), and their children aren't always other elements—sometimes they're just text. This led to two key improvements.

1.  **Props**: We introduced a `HashMap` to represent the key-value pairs of element attributes. This is a natural Rust analog to a JavaScript `props` object.
2.  **Text Nodes**: A `div` containing "Hello" needs a way to represent the string `"Hello"`. A Rust `enum` is the perfect tool for this, allowing a `Node` in our tree to be either an `Element` or `Text`.

This gave us our canonical data structures, our **Virtual DOM**:

```rust
use std::collections::HashMap;

// A Node in our VDOM tree can be one of two things
#[derive(Debug, PartialEq, Clone)]
pub enum Node {
    Element(Element),
    Text(String),
}

// An Element now has props and its children are Nodes
#[derive(Debug, PartialEq, Clone)]
pub struct Element {
    pub tag_name: String,
    pub props: HashMap<String, String>,
    pub children: Vec<Node>,
}
```

With these, we could represent a complete application structure in memory before ever touching the browser.

## Session 3: Bridging the Chasm with WebAssembly

Having an in-memory tree is great, but useless unless it can communicate with the browser. This is where **WebAssembly (Wasm)** comes in. We used `wasm-pack` to compile our Rust library into a Wasm binary and generate the necessary JavaScript "glue" code.

This process transforms our Rust library into something that looks and feels like a native npm package. To make it callable, we tagged a Rust function with an attribute: `#[wasm_bindgen]`.

The first version of our TypeScript consumer looked like this, simply proving we could call Rust and get data back:

```typescript
// examples/basic-render-test/src/main.ts
import init, { create_app_node } from 'rusty-react';

async function run() {
  // Initialize the Wasm module
  await init();

  // Call our exported Rust function!
  const appNode = create_app_node();
  console.log('Virtual DOM node from Rust:', appNode);
}

run();
```

## Session 4: From Virtual to Real - The First Renderer

This was the moment of truth. We had a VDOM tree in Rust and a bridge to JavaScript. Now, we had to write the renderer that would walk our tree and create real browser DOM nodes.

The key was the `web-sys` crate, which provides raw bindings to all browser Web APIs. This lets you write Rust code that directly calls `document.createElement()` and `parent.appendChild()`.

Our renderer is a simple recursive Rust function:

```rust
fn render_node_to_dom(v_node: &Node, document: &Document, parent: &DomNode) {
    match v_node {
        // Base case: create a text node
        Node::Text(text) => {
            let text_node = document.create_text_node(text);
            parent.append_child(&text_node).unwrap();
        }
        // Recursive step: create an element
        Node::Element(element) => {
            // 1. Create the DOM element (e.g., <div>)
            let dom_element = document.create_element(&element.tag_name).unwrap();

            // 2. Set all its attributes (props)
            for (key, value) in &element.props {
                dom_element.set_attribute(key, value).unwrap();
            }

            // 3. Append the new element to its parent
            parent.append_child(&dom_element).unwrap();

            // 4. Recurse for all children, using the new element as the parent
            for child in &element.children {
                render_node_to_dom(child, document, &dom_element);
            }
        }
    }
}
```

This is the magic. Rust is walking its own data structure and, step-by-step, building a parallel structure in the browser's DOM.

## The Inevitable Hurdles (The Fun Part)

No project is complete without debugging. We hit three classic integration issues that were fantastic learning moments:

1.  **Namespace Collision:** My own `struct Element` conflicted with the `web_sys::Element`. The Rust compiler's clear error message pointed to the solution: renaming the import with `use web_sys::{Element as DomElement}`.
2.  **Vite Security:** Vite's dev server initially blocked our app from fetching the `.wasm` file because it was outside the project's root directory. The fix was a simple one-line config change in `vite.config.ts` to adjust the `server.fs.allow` list.
3.  **The `unreachable` Panic:** Our app crashed with a `RuntimeError: unreachable`. This is the Wasm equivalent of a Rust `panic!`. Tracing it back, it came from an `.expect()` call on `document.get_element_by_id()`. The script was executing before the HTML `<div id="root"></div>` was on the page! A simple reordering of the `<script>` tag in our `index.html` fixed it.

## Where We Are and What's Next

We have a solid foundation: a Rust library that can build a virtual DOM and render it into a real, live browser DOM.

But this is just static rendering. The true magic of React lies in its name: **reactivity**. Our journey ahead involves tackling the really exciting challenges:

-   **Components:** Abstracting UI into reusable, stateful units.
-   **State Management:** Giving components a memory so they can change over time.
-   **The Reconciliation Algorithm:** This is the big one—the "diffing" engine that compares the old VDOM tree with a new one and generates a minimal list of DOM updates.

It's been an incredible journey so far. Stepping outside of established frameworks forces you to confront the fundamental problems they solve, and doing it in Rust has been a masterclass in safety, performance, and explicit design.

Stay tuned for Part 2.

#### FWI if you want to see source code for this project go to [the repo](https://github.com/mrchucu1/rusty-react).