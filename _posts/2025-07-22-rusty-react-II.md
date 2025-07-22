# Rusting the Gears of React, Part 2: Lessons in Ownership and Composition

**Posted by: Diego Navarro ([@mrchucu1](https://github.com/mrchucu1))**

In our first session, we built a static renderer. Today, our goal was to introduce the core abstraction of a UI library: the **Component**. This journey took an unexpected and valuable turn, forcing us to confront one of Rust's most important design principles—the Orphan Rule—and culminated in demonstrating the very reason components exist: **composition**.

### From A Data Tree to an Abstraction

Our initial renderer was hardcoded to a specific VDOM tree. This isn't scalable. The goal is to move towards a declarative API, like React's, where we can simply say "render the `App` component" and trust the library to figure out the rest.

To do this in Rust, we first defined a contract for all components using a `trait`. A trait is like an interface; it specifies that any `Component` must, at a minimum, know how to `render` itself into our Virtual DOM `Node` format.

```rust
pub trait Component: Debug {
    fn render(&self) -> Node;
    // ... more to come
}
```

We encapsulated our UI inside an `App` struct that implements this trait. The next logical step was to teach our VDOM about this new concept, so we added a new `Node` variant: `Component(...)`.

### The First Hurdle: Making Components Clonable

Our first attempt looked like this: `Component(Box<dyn Component>)`. This uses a `Box` (a pointer to heap-allocated data) and a `dyn Component` (a "trait object" that can hold any struct implementing our trait). This worked for the initial render, but we were thinking ahead.

To implement React's "diffing" algorithm later, we'll need to hold two VDOM trees in memory: the old one and the new one. This means we must be able to `clone` our VDOM. When we tried to automatically `derive(Clone)` on our `Node` enum, the Rust compiler correctly stopped us. A `dyn Component` trait object doesn't have a known size or structure, so the compiler has no idea how to copy it. It can't be automatically cloned.

### A Deeper Lesson from the Compiler: The Orphan Rule

Our journey to solve the `Clone` problem led us to an even more fundamental concept. Our next idea was to use `Rc` (a reference-counted smart pointer) and manually implement `Clone` for it.

```rust
// Our attempted (and illegal) code:
impl Clone for Rc<dyn Component> { /* ... */ }
```

The compiler immediately stopped us with error `E0117`, enforcing the **Orphan Rule**. This rule is a cornerstone of Rust's stability. It states: **you can't implement a foreign trait for a foreign type.**

In our case, both the `Clone` trait and the `Rc` type come from Rust's standard library; they are "foreign" to our crate. The orphan rule prevents chaos in the ecosystem by ensuring that any implementation is "owned" by either the trait's crate or the type's crate.

### The Idiomatic Solution: The "Newtype" Pattern

The solution is a powerful and common Rust pattern. If we can't implement a trait for a foreign type, we simply wrap that foreign type in a new, local type of our own. This is called a **newtype**.

We created a simple wrapper struct:

```rust
// A "newtype" that wraps the foreign `Rc` type.
// `VComponent` is OUR type, so we can implement traits for it.
#[derive(Clone)]
pub struct VComponent(Rc<dyn Component>);
```

Because `VComponent` is local to our crate, we were free to implement `Clone` for it, satisfying the orphan rule. This experience was a perfect demonstration of Rust's philosophy: the compiler's strictness isn't a hindrance; it's a guide that pushes us toward better, safer, and more robust designs.

### The Payoff: Composition in Action

With our robust and efficient component architecture in place, it was time to prove its value. We created a new, simple component called `Greeter` whose job is to display a message passed to it via a struct field (our version of "props").

```rust
#[derive(Clone)]
pub struct Greeter {
    pub message: String, // This field acts like a React "prop"
}

impl Component for Greeter {
    fn render(&self) -> Node {
        // Renders an <h2> with the message
        // ...
    }
    // ...
}
```

This `Greeter` is a reusable, self-contained piece of UI. The true power was revealed when we modified our main `App` component to *use* it. The `render` method for `App` now looks like a real UI component: it renders some of its own elements and **composes** them with instances of other components.

```rust
// Inside App's render method...
fn render(&self) -> Node {
    Node::Element(Element {
        // ...
        children: vec![
            Node::Element(/* a static h1 element */),

            // Use the Greeter component!
            Node::Component(VComponent(Rc::new(Greeter {
                message: "This is a composed component...".to_string(),
            }))),

            // Reuse the Greeter component with different data!
            Node::Component(VComponent(Rc::new(Greeter {
                message: "Here is another, demonstrating reusability!".to_string(),
            }))),
        ]
    })
}
```

When you build and run the example now, you'll see the output from both the `App` and the `Greeter` components rendered to the screen. This is the heart of component-based architecture. We've successfully created an abstraction that allows us to build complex UIs from small, independent, and reusable pieces.

This solid foundation of clonable, composable components is exactly what we need for our next and most exciting challenge: state management and reactivity.