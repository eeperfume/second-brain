---
aliases: [검증 가능한 React 컴포넌트 원문 en, Verifiable React Components]
---

# Verifiable React Components

This is an idea file. It's meant to be handed to a capable coding agent working
inside your React project — Claude Code, Codex, whatever you use — and read once.
It isn't a library; there's nothing to install. Its only job is to land one idea
clearly enough that the agent can build, together with you, a verification system
that fits your stack. The idea is what travels between projects. The code gets
written fresh each time, for the project it lives in.

## The problem

Start with a plain question: how do you actually know a React component works?

The usual answers all leave something out. Unit tests reach inside the component
and assert on its state and props, so they keep passing right up until you
refactor — and then they break even though nothing a user can see has changed.
Snapshot tests freeze the markup and fail on every harmless tweak. Clicking
through by hand proves it worked once, on your machine, for the one path you
happened to try. And "looks fine to me" proves nothing at all. Worse, each of
these answers is walled off from the others: the test runner has its own verdict,
you have a different impression from clicking around, and an AI agent trying to
check the work has no way in. Three judges, three answers, and no way to reconcile
them.

What they have in common is that they inspect the *implementation* — the wiring
inside the box — instead of what anyone outside the box can observe.

## The idea

Turn it around. Have each component **report its own state to the surface** — the
rendered DOM — in plain data attributes. Then everyone judges the component by
reading that same surface: the person watching it, the CI job gating the merge,
the AI agent inspecting the work. The surface becomes the single shared answer to
"does this work."

Concretely, a component renders something like this:

```html
<section
  data-verify-unit="CartSummary"
  data-verify-item-count="3"
  data-verify-subtotal-cents="4997"
  data-verify-empty="false"
>
```

That's the component saying, out loud and in the open, what it believes is true
about itself right now. You can read it without knowing a thing about React — no
hooks, no fibers, no guessing from class names. Those attributes are its gauges;
you read the gauges, not the wiring behind them. The values are just strings,
compared as text, so the same value has to come out as the same string — which is why the price is in
whole cents, not a float that might render two ways.

This buys a lot. The DOM is what the user actually gets, so you're checking the
real output rather than a stand-in for it. The same readout is legible to a test,
a person, and an agent alike, so the three can finally agree. And the internals are
free to change — rewrite the hooks, swap the state library — as long as the surface
keeps telling the truth. The one thing a component may not do is stay silent: a
component that reports nothing can't be verified, so we treat that as a failure.

## The engine and the specs

There are two parts, and keeping the line between them clean is what makes the
whole thing portable.

The **engine** is the reusable machinery. It mounts a component on its own in a
known state, drives whatever interaction the case calls for, reads the surface,
runs the checks, and gathers all of it into a single verdict. There's no engine
here to copy, though — it's described only by what it has to *do*, so your agent
writes it natively for your test runner, your router, your schema library. Rules,
not code.

The checks the engine runs aren't all of one kind. Beyond asking whether the
surface matches what the spec declared, some cut across every component — whether
the data-verify-* attributes are all present, whether the rendered view is
accessible, whether it stays within a performance budget (bundle under 200KB,
first paint under 2.5s, no more than 50 requests) — and these live with the
engine, not the components. So you can add a new kind of check, whatever the
project needs, without touching a single component.

The **specs** are the only thing you write per component, and there isn't much to
them. To verify a component, you name a few of the states it can be in — an empty
cart, one item, a hundred items — say what drives it into each, and write down what
must be true in that state: the subtotal equals the sum of the line items, an empty
cart reports `empty=true`. The component emits the data attributes on its own. To
verify another component you write one more short spec, and the engine already knows
what to do with it.

But a spec is only as good as the states you check and the values you expose; choose
wrong and a bug sails through. So two things matter. First, pick a state where a
correct component and a buggy one actually differ. Say there's a 10%-discount bug
that truncates the discount instead of rounding it. At a price of 1000 cents the
discount is exactly 100, so round and truncate both leave a subtotal of 900 and the
bug stays hidden. It only surfaces at a price with an awkward remainder: at 999
cents the discount is 99.9, which the correct code rounds to 100 (subtotal 899) and
the buggy code truncates to 99 (subtotal 900) — a one-cent split that finally tells
them apart. The happy path alone proves little. Second, put a gauge on whatever could break, because a value
the component never puts on the surface is one no check can ever read.

## The verdict

A run ends in one of four words, and what separates them is the whole point.

**PASS** and **FAIL** are the obvious two: it was checked and held, or it was
checked and didn't.

**BLOCKED** is the one people forget, and the most useful. It means the system
couldn't even look — the component never mounted, the setup threw, part of the
page hadn't finished loading. That is not the same as FAIL, and the moment you lump
the two together, a real failure can hide among the false alarms. BLOCKED says "no
verdict was possible here," and it asks to be looked at rather than waved through
as success or failure.

**SKIP** is for when there was genuinely nothing to check.

Underneath all of it sits one bias: **when you're not sure, fail.** A false pass
ships a broken component to real users, and pays dearly when it breaks. A false fail
just makes someone look twice — cheap by comparison. The costs are that lopsided, so
you lean to the cheap side. That same instinct is why
anything the system couldn't observe comes back BLOCKED rather than PASS. If you
didn't see it work, you don't get to call it working.

## Proving it catches lies

Here is the part that makes the whole thing trustworthy, and the part most testing
skips.

A suite that has only ever gone green has proven nothing. Maybe it's vigilant;
maybe it can't tell good from bad and would go green no matter what. From the
outside you can't know which. So the system has to demonstrate that it can say no.

Two rules do it. First, every component carries at least one case off the happy
path — an input that should be rejected, an edge that usually breaks things.
Second, somewhere in the project at least one case is **built to fail**, and the
suite explicitly asserts that it *does* fail — its failing is the healthy outcome,
so you check for it but don't let it drag the whole run red. That second rule is
the load-bearing one: it tests the checker itself, not the component. The day someone loosens a
check until the broken case slips through, the "this must fail" assertion goes
green-on-broken and the suite breaks loudly, on purpose. The system earns trust by
proving, over and over, that it still catches the lie it was set up to catch.

## Compute the verdict once

Compute the verdict once, write it down, and let everyone read the same record.

The human's dashboard, the CI gate, and the agent's machine-readable handle are
all just views onto that one stored result. None of the three recomputes it. This is what
makes "green in CI but the agent sees red" impossible: there is exactly one answer,
and the three are reading the same copy. When the place that computes the verdict
and the place that displays it are different — a test runner produces the result, a
page shows it later, say CI verifying before a merge while a status page in the app
reads and shows that result — the written result is the handoff. The page renders what was
computed; it doesn't run the checks again and risk a different story.

For the agent specifically, leave the answer somewhere it can read without running
code: a node on the page when the app is running in a browser, a results file when
it isn't (a CI-only run, say).
But the word alone is a label, not a repair kit. For every check that didn't hold,
the record has to carry enough to act on without re-running anything — which unit,
what it expected, what the surface actually reported — because an agent handed only
"FAIL" learns that something broke, never what or where.

## Function vs. appearance

Whether a component does what it's supposed to (is-right) and whether it merely
looks right (looks-right) are two different questions, and the system should never
confuse them.

Whether it works is the machine's job: read the surface, check the rules,
headlessly, and that is what produces the verdict. Whether it looks right is for
human eyes — a clean render of each case you can glance at or screenshot. It's worth
having, but it isn't the verdict, and it only exists where there's actually
something to look at: a real browser with real pixels. Run your checks in a
headless environment and there are no pixels at all. That's fine; you just don't
pretend the visual pass happened. Never let looks-right stand in for is-right.

## Adapting it to your stack

Everything above assumes you can take a component, mount it by itself, and read its
surface. In a plain client-rendered app that's straightforward: a hidden route that
renders one component in a chosen state does the job.

It gets more interesting when some of the work happens on the server. Modern React
frameworks split a page across a server boundary — Next.js renders some components on
the server and sends them down, Remix fetches and changes data with server functions
called loaders and actions — and the rule is the same in every case: **the surface is
whatever ends up in the browser, and that's the only place you read it from.**

Server work that never reaches the browser can't be judged from the surface, so don't
try. The fetch, the data shaping, the part of a server component or a loader that
decides what the UI gets — pull that logic into a plain function and verify it
directly, or honestly mark it unverified. You can't get at it by mounting, either: a
server component won't run in a client test harness, and a loader is just a function,
not a component. So mount only what reaches the browser. If you need a server-rendered
surface itself, read it from the page actually running at its route, not from an
isolated test mount.

And don't confuse the two: a component that renders correctly given some data tells
you the UI is right, not that the server produced the right data. Verify that part
separately, as the plain function above.

The other wrinkle is timing. Anything that resolves a beat after you ask for it has
to settle before you read — a form submission the server handles and then
re-renders from, a route's data loading in for the first time, or just a
client-side fetch. So drive the interaction, wait for it to settle, then read the
surface.

You'll meet the same shape of decision in any stack — what the surface is, how you
mount one piece in isolation, where the answer gets written. Answer those for the
project in front of you.

## A note

This document is deliberately abstract. The exact attribute
names, the test runner, how you mount a
component, where the result gets written, how you handle a list that only renders
what's on screen — all of that depends on your project, and your agent can work it
out. (That last one looks small but it bites: a windowed list still has to report
its totals from its own state, not by counting the rows that happen to be on
screen, or you'll cheerfully verify a lie.)

Hand it to your coding agent, talk through your stack, and build the version that
fits. You'll know it landed when components publish their own state, when humans and CI
and agents all read the same verdict, when the system can prove it still catches a
deliberately broken case, and when nothing that went unobserved gets called a pass.
The agent can take it from there.
