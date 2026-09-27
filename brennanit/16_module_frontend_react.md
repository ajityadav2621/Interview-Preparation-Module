# Module 6: Frontend — React / Angular / Redux

---

## PART 1: Fundamentals (must know cold)

- **Component-based UI**: the UI is broken into reusable, composable components, each managing its own rendering logic.
- **Props vs state**: props are passed *into* a component from its parent (read-only from the child's perspective); state is data a component manages internally and can change over time, triggering a re-render.
- **Unidirectional data flow**: data flows down via props, events flow up via callbacks — a child never directly mutates a parent's state, it calls a function the parent passed down.
- **Hooks**: `useState` (local state), `useEffect` (side effects — runs after render, re-runs when dependencies change), `useMemo`/`useCallback` (memoization to avoid unnecessary recomputation/re-renders).
- **Virtual DOM**: React builds an in-memory representation of the UI, diffs it against the previous version on state change, and applies only the minimal necessary changes to the real DOM.
- **Redux core loop**: action (a plain object describing "what happened") → reducer (a pure function computing new state from old state + action) → store (holds the single source of truth state) → component re-renders when the relevant slice of state changes.
- **Controlled vs uncontrolled components**: controlled = form input value is driven by React state (`value` + `onChange`); uncontrolled = the DOM manages its own value internally (accessed via a ref when needed).

---

## PART 2: Interview Questions & Answers

**Q1: What's the difference between props and state, and why does that distinction matter?**
> Props come from the parent and a component shouldn't modify them directly — they represent configuration/data passed down. State is local and owned by the component itself, and changing it (via `setState`/the state setter from `useState`) is what triggers a re-render. The distinction matters because it enforces a predictable, one-directional flow of data — if a component could freely mutate its own props, you'd lose the ability to reason about where a piece of data is "owned" and where its true source of truth lives.

**Q2: When would you lift state up versus keep it local to a component?**
> If only one component needs a piece of state, I keep it local with `useState` — no reason to complicate things. I lift it up to the nearest common parent (or into Redux/context) only when two or more sibling components need to share or react to the same state — for example, a filter control and a results list that both need the current filter value.

**Q3: Why would a `useEffect` cause an infinite re-render loop, and how do you avoid it?**
> This usually happens when the effect updates state that's also in its own dependency array without a proper guard — the state update triggers a re-render, which re-runs the effect (since its dependency changed), which updates the state again, looping forever. I'd avoid it by making sure the dependency array only includes values that should actually trigger a re-run, and that any state update inside the effect is conditional (e.g., only update if the new value is actually different) or restructure the logic so the effect doesn't need to depend on the value it's setting.

**Q4: What's the point of `React.memo` or `useMemo`, and when would you actually reach for them?**
> They prevent unnecessary recomputation/re-renders — `useMemo` caches an expensive computed value between renders unless its dependencies change, and `React.memo` skips re-rendering a component if its props haven't changed. I'd reach for them only after profiling shows an actual performance cost from re-renders/recomputation — adding memoization everywhere by default adds complexity and a small overhead of its own without a guaranteed benefit.

**Q5: How does Redux help versus just using component state everywhere?**
> Once several unrelated parts of the UI need to read or update the same piece of state, passing it down through props at every level ("prop drilling") gets unwieldy. Redux gives you one central store as the single source of truth, and any component can subscribe to just the slice of state it needs, without needing to know about the components in between. I'd avoid pulling everything into Redux by default though — state that's genuinely local to one component or a small subtree usually doesn't need to be global.

---

## PART 3: How It Works Internally

**How the Virtual DOM diffing/reconciliation actually works**: on a state change, React re-runs the component's render function to produce a new virtual DOM tree (a lightweight JS object representation, not real DOM nodes). It then diffs this against the previous virtual DOM tree using a heuristic algorithm (comparing elements by type and position, using `key` props to track list items across re-renders) to compute the minimal set of real DOM mutations needed, then applies only those — this is what makes React fast despite re-rendering component functions frequently, since the expensive part (real DOM manipulation) is minimized, not the cheap part (running JS functions).

**Why `key` matters in lists internally**: without a stable `key`, React's diffing falls back to comparing elements by position/index, which can cause it to incorrectly reuse/mutate the wrong DOM node when items are reordered, inserted, or removed (e.g., input focus jumping to the wrong row). A stable, unique `key` lets React's reconciler correctly match each virtual DOM element to the same real DOM node across renders, even if its position in the list changed.

**How `useEffect`'s dependency array actually triggers re-runs**: React stores the dependency array from the previous render and does a shallow comparison (`Object.is` per element) against the new render's dependency array. If any value differs, the effect's cleanup function (if provided) runs first, then the effect itself re-runs. This is why passing a new object/array/function literal as a dependency on every render (instead of a memoized one) causes the effect to re-run every single render — the shallow comparison sees a "different" reference even if the contents are logically the same.

**How Redux's store and subscriptions work internally**: the store holds a single state tree and a list of subscriber callbacks. Dispatching an action calls the root reducer (a pure function) with the current state and the action, producing a new state object; the store then notifies all subscribers that state changed. Libraries like `react-redux` optimize this so a component only re-renders if the *specific slice* of state it selected (via `useSelector`) actually changed — comparing the selected value before and after, not just "did the store change at all" — which is why writing selectors that return primitives or memoized derived values (rather than a new object literal every time) matters for performance.
