# Hotwire Native Bridge Components for Complex Inputs
## A working guide for Rails → iOS & Android

Twenty-one chapters covering the complex form inputs: eleven complete bridge components with working Rails, JavaScript, Swift and Kotlin, and two chapters with no native code at all — Chapter 8, where HTML attributes are the answer, and Chapter 13, where Turbo Frames are. **Chapter 20 collects twelve upstream issues** whose symptoms point somewhere other than their cause; read it once, then return to it when something inexplicable happens. Device capabilities — upload, camera, signature, location, rich text — are outside this guide's scope, and Appendix B says what they require.

**The architecture this guide assumes.** One Rails app serving a complete, browser-first website to everyone. Forms are ordinary Rails forms and they work in a plain browser. The native iOS and Android apps load those same URLs through Hotwire Native, and only *parts* of some pages are swapped for native-aware versions. The site is the product; native is a set of swap points on top of it. Chapter 3 covers the server side of that arrangement; everything after it is the bridge components themselves.

---

## Target versions

| Component | Version |
|---|---|
| `@hotwired/hotwire-native-bridge` | **1.2.2** |
| `hotwire-native-ios` | **1.3.0** |
| `hotwire-native-android` | **1.3.1** |
| `turbo-rails` | current |
| Rails | 7.1+ |

These libraries move, and API details in this guide may be behind or ahead of what you have installed. Check anything surprising against the source of the version you are actually running before building on it. **Appendix A** records the API surface this guide was written against.

Four things people commonly get wrong:

- iOS has **both** `reply(to:)` and `reply(to:with:)` — the second takes any `Encodable`.
- Android's generic parameter is **`HotwireDestination`**, not the older `BridgeDestination`.
- The JS callback receives a **message object**, so you read `message.data.yourField`, not `data.yourField`.
- Rails detection is `hotwire_native_app?`, which matches `/(Turbo|Hotwire) Native/` — it ships in `turbo-rails`, so you don't write it yourself.

---

## How to read this

**If you are a person:** read Chapters 1 and 3, skim 2 and 4, then jump to whichever component you need. Chapters 5–17 are independent of each other. If you only read one thing, make it Chapter 3 — most "we need a bridge component" turns out to be a variant partial or three lines of CSS.

**The chapters are ordered by complexity**, and they are grouped: 1–4 foundations, 5–8 the common cases, 9–12 selection and dates, 13–17 the complex form patterns, 18–21 production concerns, debugging, and field notes.

**If you are an agent:** Chapters 1–4 are prerequisites for every component chapter. Component chapters have a fixed structure — *the web problem · the contract · the code (Rails → JS → CSS → Swift → Kotlin) · wiring · verification · gotchas*. Every code block is complete; there are no elisions and no `// ...`. Each block's first line is a path comment telling you where to write it.

**Conventions used throughout:**

- `⚠️` marks something version-sensitive or dependent on behavior that may differ across platforms and releases.
- Package names assume a Rails app with importmap, an iOS app with the component files under `Bridge/`, and an Android app under `dev.hotwire.demo.bridge`. Adjust paths to your own layout; nothing else changes.
- Dates on the wire are always **ISO-8601 strings in UTC**. Chapter 10 explains why this is not negotiable.

---

# Chapter 1 — The model

## 1.1 The one rule

> **Form state, validation, and persistence stay in Rails. Only the editing affordance moves native.**

Every decision in this guide follows from that. A native date picker does not know your validation rules, does not know your timezone policy, and must not become a second source of truth. It is a nicer way to produce a string that goes into the same form field the web would have used.

This is what makes the whole approach survivable: **the browser form is the real form.** The native component is a better way to fill it in. When the native component is absent — a browser, an old app version, a platform you haven't shipped yet — the form still works, because nothing essential ever lived in Swift or Kotlin.

Teams that violate this rule end up maintaining three validation implementations and shipping App Store releases to fix business logic. Teams that hold it ship web changes on Friday.

## 1.2 What "browser-first" costs you, and what it buys

Building browser-first means accepting one constraint: **every feature must be designed to work without any native code at all, and must then be improved by native code rather than completed by it.**

That is a real constraint. It rules out designs where the native component owns a step of the flow. It means the native app can never be the only place a thing can be done.

What it buys is proportionate:

| | Browser-first + swap points | Native-owned screens |
|---|---|---|
| Shipping a fix | Deploy Rails | App-store review, then wait for adoption |
| Users on the old app version | Get the web version of the change | Get the old behavior until they update |
| Surfaces to maintain | One, plus a few swap points | Two or three, permanently |
| New feature cost | Web only, natively enhanced later | Web plus two native implementations |
| Where bugs live | Mostly one codebase | Spread across three |

The version-skew row is the one that decides it for most teams. In a browser-first app, a user on a six-month-old build still gets today's business logic — they just get the web control instead of the native one. In a native-owned app, they get six-month-old behavior, and you cannot make them update.

Chapter 3 turns this into concrete Rails mechanics. The rest of this chapter is about the bridge itself.

## 1.3 What a bridge component actually is

Three objects with the same name, talking over a JSON channel:

```
┌────────────────────────────┐
│  Rails view (ERB)          │
│  data-controller=          │
│   "bridge--date-picker"    │
└──────────┬─────────────────┘
           │ Stimulus connects
           ▼
┌────────────────────────────┐
│  Stimulus BridgeComponent  │
│  static component =        │
│    "date-picker"      ─────┼──── the name that must match ────┐
│                            │                                  │
│  this.send(event, data,    │                                  │
│            callback)       │                                  │
└──────────┬─────────────────┘                                  │
           │ JSON over the WebView message handler              │
           ▼                                                    ▼
┌────────────────────────────┐          ┌─────────────────────────────┐
│  Swift BridgeComponent     │    or    │  Kotlin BridgeComponent     │
│  class var name =          │          │  BridgeComponentFactory(    │
│    "date-picker"           │          │    "date-picker", ::Comp)   │
│                            │          │                             │
│  onReceive(message:)       │          │  onReceive(message)         │
│  reply(to:with:)           │          │  replyTo(event, data)       │
└────────────────────────────┘          └─────────────────────────────┘
```

The string `"date-picker"` appearing in all three places is the entire contract. Get it wrong and you get silence — no error, no warning, nothing happens. That is the single most common failure and Chapter 19 covers how to spot it in two minutes.

## 1.4 Anatomy of a message

A message has three fields you care about:

| Field | Set by | Purpose |
|---|---|---|
| `component` | the library | routes to the right native class |
| `event` | you | which operation — `"connect"`, `"display"`, `"submitEnabled"` |
| `data` | you | the JSON payload |

The flow in both directions:

```
Web                                     Native
 │                                        │
 │  send("display", { title, items },     │
 │        callback)                       │
 ├───────────────────────────────────────▶│  onReceive(message)
 │                                        │    message.event == "display"
 │                                        │    message.data<MessageData>()
 │                                        │
 │                                        │  …user taps something…
 │                                        │
 │  callback(message)                     │  replyTo("display", { index: 2 })
 │◀───────────────────────────────────────┤  reply(to: "display", with: …)
 │  message.data.index === 2              │
 │                                        │
```

Three properties of this channel that shape how you design components:

1. **The reply is keyed by event name, not by call.** `reply(to: "display", …)` fires the callback registered by the most recent `send("display", …)`. Two in-flight `display` messages will confuse each other. Design one conversation at a time per event.
2. **A reply is optional and may never come.** If the user cancels a native sheet, most implementations simply don't reply. The JS callback is then never invoked. That is usually fine — but if your JS is holding a lock, a spinner, or a disabled button waiting on it, you have a hang. Always ask "what if the reply never arrives?"
3. **The native side can reply more than once.** `reply(to:)` can fire repeatedly for the same `send`. The submit button in Chapter 5 relies on exactly that: one `connect` message, a reply on every tap, forever.

---

# Chapter 2 — Setup

## 2.1 Web

```sh
./bin/importmap pin @hotwired/stimulus @hotwired/hotwire-native-bridge
```

Import order matters, and this is the web-side twin of the registration-order problem above:

```js
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "@hotwired/hotwire-native-bridge"   // creates the bridge object
import "controllers"                        // controllers that extend BridgeComponent
```

Importing the bridge library *after* your controllers means those controllers have nothing to attach to, and their `initialize` and `connect` never fire — while an ordinary Stimulus controller on the same element works perfectly. That asymmetry is the diagnostic: **swap `BridgeComponent` for a plain `Controller` and see whether lifecycle events fire.** If they do, the problem is the bridge, not Stimulus. See 20.2 entry 2.

Bridge controllers go in their own namespace so they are obviously different from ordinary Stimulus controllers:

```
app/javascript/controllers/
├── application.js
├── index.js
└── bridge/
    ├── form_controller.js            → identifier "bridge--form"
    ├── toast_controller.js           → identifier "bridge--toast"
    ├── confirm_controller.js         → identifier "bridge--confirm"
    ├── select_controller.js          → identifier "bridge--select"
    ├── date_picker_controller.js     → identifier "bridge--date-picker"
    ├── multi_select_controller.js    → identifier "bridge--multi-select"
    ├── date_range_controller.js      → identifier "bridge--date-range"
    ├── typeahead_controller.js       → identifier "bridge--typeahead"
    ├── nested_form_controller.js     → identifier "bridge--nested-form"
    ├── wizard_controller.js          → identifier "bridge--wizard"
    └── unsaved_changes_controller.js → identifier "bridge--unsaved-changes"
```

With `stimulus-loading`'s eager loading, a file at `controllers/bridge/date_picker_controller.js` registers as `bridge--date-picker`. That identifier is what goes in `data-controller`.

> **The Stimulus identifier and the component name are different things.** `bridge--date-picker` is how Stimulus finds the controller. `static component = "date-picker"` is how the native app finds it. They do not need to match each other, but the second one must match the native registration exactly.

## 2.2 iOS

Components live in `Bridge/` and are registered before any navigation happens:

```swift
// ios/MyApp/AppDelegate.swift
import HotwireNative
import UIKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {
        Hotwire.registerBridgeComponents([
            FormComponent.self,
            ToastComponent.self,
            ConfirmComponent.self,
            SelectComponent.self,
            DatePickerComponent.self,
            MultiSelectComponent.self,
            DateRangeComponent.self,
            TypeaheadComponent.self,
            NestedFormComponent.self,
            WizardComponent.self,
            UnsavedChangesComponent.self
        ])

        return true
    }
}
```

Registration must happen **before** any `Navigator` or `Session` is created or accessed. Registering late is a silent failure with no error of any kind.

> **The precise mechanism, because "before the first Session" is not quite the whole story.** The registration list is appended to the WebView's user agent string, and on iOS that string is built **the first time the `Navigator`'s `rootViewController` is accessed**. A setup that merely *reads* `navigator.rootViewController` before calling `registerBridgeComponents` — which some published examples did — produces a user agent with an empty component list, and every component in the app reports `this.enabled == false`. Registering first is not a style preference. See 20.2 entry 1.

## 2.3 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/DemoApplication.kt
package dev.hotwire.demo

import android.app.Application
import dev.hotwire.core.bridge.BridgeComponentFactory
import dev.hotwire.core.config.Hotwire
import dev.hotwire.demo.bridge.ConfirmComponent
import dev.hotwire.demo.bridge.DatePickerComponent
import dev.hotwire.demo.bridge.DateRangeComponent
import dev.hotwire.demo.bridge.FormComponent
import dev.hotwire.demo.bridge.MultiSelectComponent
import dev.hotwire.demo.bridge.NestedFormComponent
import dev.hotwire.demo.bridge.SelectComponent
import dev.hotwire.demo.bridge.ToastComponent
import dev.hotwire.demo.bridge.TypeaheadComponent
import dev.hotwire.demo.bridge.UnsavedChangesComponent
import dev.hotwire.demo.bridge.WizardComponent

class DemoApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        Hotwire.registerBridgeComponents(
            BridgeComponentFactory("form", ::FormComponent),
            BridgeComponentFactory("toast", ::ToastComponent),
            BridgeComponentFactory("confirm", ::ConfirmComponent),
            BridgeComponentFactory("select", ::SelectComponent),
            BridgeComponentFactory("date-picker", ::DatePickerComponent),
            BridgeComponentFactory("multi-select", ::MultiSelectComponent),
            BridgeComponentFactory("date-range", ::DateRangeComponent),
            BridgeComponentFactory("typeahead", ::TypeaheadComponent),
            BridgeComponentFactory("nested-form", ::NestedFormComponent),
            BridgeComponentFactory("wizard", ::WizardComponent),
            BridgeComponentFactory("unsaved-changes", ::UnsavedChangesComponent)
        )
    }
}
```

The string in `BridgeComponentFactory` is the component name. It is passed to your component's `name` constructor parameter, so you never hardcode it twice.

You also need kotlinx.serialization, since `message.data<T>()` decodes through it:

```kotlin
// android/app/build.gradle.kts
plugins {
    kotlin("plugin.serialization") version "2.0.21"
}

dependencies {
    // The compiler plugin above generates serializers; this is the runtime
    // that reads and writes them. Omitting it compiles and fails at runtime.
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
}
```

## 2.4 Platform minimums

The code in this guide is not version-neutral, and two of its dependencies are easy to miss until a build fails on someone else's machine.

**iOS 16.0.** `hotwire-native-ios` itself targets lower, but this guide's components use:

| API | Available from | Used in |
|---|---|---|
| `UIBarButtonItem(systemItem:primaryAction:)` | iOS 14 | 5, 10, 11, 12 |
| `UIDatePicker.preferredDatePickerStyle = .inline` | iOS 14 | 10 |
| `sheetPresentationController`, `.detents` | iOS 15 | 10, 12 |
| `String(localized:)` | iOS 15 | 11, 14, 16 |
| `UIBarButtonItem.flexibleSpace()` | **iOS 16** | 15 |

`flexibleSpace()` is the binding constraint. On a lower deployment target, replace it with `UIBarButtonItem(barButtonSystemItem: .flexibleSpace, target: nil, action: nil)` and the rest drops to iOS 15.

**Android: `java.time` needs API 26, or desugaring.** Chapters 10 and 12 use `LocalDate`, `Instant` and `ZoneOffset` to keep date handling in UTC. Those are API 26+. If your `minSdk` is lower — and plenty of production apps are still at 24 — enable core library desugaring rather than rewriting the date code with `Calendar`:

```kotlin
// android/app/build.gradle.kts
android {
    compileOptions {
        isCoreLibraryDesugaringEnabled = true
    }
}

dependencies {
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.1.2")
}
```

Without it, the date components compile and then throw `NoClassDefFoundError` on devices below API 26 — a crash that never appears on a modern emulator.

---

# Chapter 3 — Serving native from a browser-first Rails app

*Complexity 2 · Rails only, no native code*

## 3.1 The architecture this guide assumes

One Rails app. It serves the full site to every browser — desktop, tablet, phone. **That site is the product.** It is not a degraded fallback and it is not a shell wrapped around an app. Forms are ordinary Rails forms; they are written browser-first and they work browser-first.

The native apps are Hotwire Native wrappers loading those same URLs. The only difference is that **parts** of some pages are swapped for native-aware versions when a native app is the one asking.

That word "parts" is the whole architecture. You are not building a native version of the site. You are building a browser site with a handful of swap points.

## 3.2 Two questions, not one ladder

Making a page native-aware involves two separate decisions. Treating them as one is the most common source of confusion in this architecture, so they get separate tables.

**Question 1 — what mechanism does the behavior need?**

| Mechanism | Cost | Use when |
|---|---|---|
| **Plain HTML** — `inputmode`, `autocomplete`, `enterkeyhint` | Free | The platform already does it. Chapter 8 |
| **Different markup for native** | One extra template | The native page needs a different arrangement, less chrome, or elements a browser has no use for |
| **A bridge component** | Three implementations plus a store review | Native UI or a device capability is genuinely required |

**Question 2 — how does that markup reach the page?**

| Delivery | Cost | Use when |
|---|---|---|
| **Shared template** — one partial for everyone, CSS hides what native replaces | Free | The native wiring is a few inert data attributes on elements that already exist |
| **Variant partial** — `_field.html+hotwire_native.erb` | One extra template | Native needs elements a browser shouldn't receive, or the browser needs an enhancement native shouldn't receive |

**The two questions are independent, and the answers compose.** A bridge component delivered through a variant partial is not a contradiction — it is frequently the right answer, and it is what most real forms converge on once they have more than one or two native-aware fields. 3.5 covers that pattern specifically, because it is the one most likely to be missed.

The escalation that does matter lives entirely in question 1: try plain HTML, then a server-rendered difference, then a bridge component. Reaching for a bridge component without asking whether an HTML attribute or a different partial would do is the expensive mistake — not choosing to deliver that component through a variant.

A useful sanity check: if most fields on a form have bridge components, something has gone the wrong way. On a typical form, most fields need nothing at all.

## 3.3 Detecting a native request

Hotwire Native appends a marker to the WebView's user agent automatically. You configure only your own prefix:

```swift
// ios/MyApp/AppDelegate.swift
Hotwire.config.applicationUserAgentPrefix = "MyApp/\(Bundle.main.appVersion);"
```

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/DemoApplication.kt
Hotwire.config.applicationUserAgentPrefix = "MyApp/${BuildConfig.VERSION_NAME};"
```

The library appends the rest, producing a user agent shaped like this:

```text
MyApp/3.4.1; Hotwire Native iOS; Turbo Native iOS; bridge-components: [form toast confirm select date-picker multi-select date-range typeahead nested-form wizard unsaved-changes]; Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X) AppleWebKit/605.1.15 …
```

Three things are in there, and each has a different job:

| Fragment | Who reads it | What it's for |
|---|---|---|
| `MyApp/3.4.1;` | you | your own version gating |
| `Hotwire Native iOS; Turbo Native iOS;` | Rails | native detection and platform |
| `bridge-components: [...]` | the bridge JS | populating `data-bridge-components` on `<html>` |

`turbo-rails` ships the detection, so you don't write it:

```ruby
# turbo-rails — app/controllers/turbo/native/navigation.rb
def hotwire_native_app?
  request.user_agent.to_s.match?(/(Turbo|Hotwire) Native/)
end
```

It is available as a controller method **and** as a view helper, with `turbo_native_app?` as an alias. The `Turbo|Hotwire` alternation is what keeps apps built against the older Turbo Native libraries working.

Platform detection is not built in. Add it once:

```ruby
# app/controllers/concerns/native_platform.rb
module NativePlatform
  extend ActiveSupport::Concern

  included do
    helper_method :hotwire_native_platform,
                  :hotwire_native_ios?,
                  :hotwire_native_android?,
                  :hotwire_native_version
  end

  private

  def hotwire_native_platform
    return nil unless hotwire_native_app?

    case request.user_agent.to_s
    when /(Turbo|Hotwire) Native iOS/ then :ios
    when /(Turbo|Hotwire) Native Android/ then :android
    end
  end

  def hotwire_native_ios?
    hotwire_native_platform == :ios
  end

  def hotwire_native_android?
    hotwire_native_platform == :android
  end

  # Reads the prefix you configured, e.g. "MyApp/3.4.1;" => "3.4.1"
  def hotwire_native_version
    request.user_agent.to_s[%r{MyApp/([\d.]+)}, 1]
  end
end
```

> **Don't route business decisions through the user agent.** Use it for presentation and for version gating. Anything that affects what gets saved belongs in the model, where it applies to every client equally. See 3.6.

## 3.4 Variants

Rails' variant mechanism is what makes a native-specific template possible at all, and it costs one `before_action`:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include NativePlatform

  before_action :set_hotwire_native_variant

  private

  def set_hotwire_native_variant
    return unless hotwire_native_app?

    # Rails resolves an array of variants in order, so a platform-specific
    # template wins when one exists and the generic one is the fallback.
    # ⚠️ Confirm this ordering against your Rails version with a two-template
    # test before you depend on it — it is a resolver detail, not a public
    # contract, and it is easy to verify and awkward to debug.
    request.variant = [:"hotwire_native_#{hotwire_native_platform}", :hotwire_native]
  end
end
```

Then templates opt in by existing:

```text
app/views/work_orders/
├── _form.html.erb                          ← every browser, and native by default
├── _form.html+hotwire_native.erb           ← all native apps
├── _filters.html.erb
├── _filters.html+hotwire_native_ios.erb    ← iOS only
├── index.html.erb
└── show.html.erb
```

**Rails falls back to the plain template whenever a variant doesn't exist.** That is the property that makes this safe: you add variants only where you need them, and everything else keeps serving one template to everyone. There is no parallel view tree to maintain.

Layouts work the same way, and this is usually the first variant a project needs — the native app draws its own navigation, so the web chrome has to go:

```ruby
# app/controllers/application_controller.rb
layout -> { hotwire_native_app? ? "hotwire_native" : "application" }
```

```erb
<%# app/views/layouts/hotwire_native.html.erb %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for(:title) || "MyApp" %></title>
    <meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body class="native">
    <%# No site header, no footer, no cookie banner — the app supplies navigation. %>
    <main><%= yield %></main>
  </body>
</html>
```

> **Keep the tracked asset set identical to the application layout.** `data-turbo-track="reload"` asset lists must be **invariant across every navigable page**. A second layout is precisely where they drift — someone adds a page-specific bundle to one layout and not the other — and the consequence is not a missing stylesheet. It is a screen that hangs on a loading spinner during back navigation, because `pageInvalidated` fires on a restore visit, cancels it, cold-boots, and never dismisses the loading overlay. 20.3 entry 7 has the full diagnosis. Track only truly global bundles here, and track the same ones you track in `application.html.erb`.

## 3.5 Delivering bridge components through partial variants

A bridge component is markup plus JavaScript plus native code. The native code is registered at app launch and the JavaScript is in your bundle — but the **markup has to get onto the page**, and there are two ways to put it there.

### Inline in the shared partial

The field everyone already sees gains a few data attributes:

```erb
<%# app/views/work_orders/_form.html.erb — one template, all audiences %>
<div data-controller="bridge--date-picker"
     data-bridge--date-picker-title-value="Scheduled date">
  <%= form.date_field :scheduled_on,
        data: { "bridge--date-picker-target": "input" } %>
</div>
```

Browsers receive attributes they ignore. Nothing extra is rendered, nothing needs hiding, and there is one template. Chapters 9, 10 and 12 work this way, and for fields like these it is the right call — the native version enhances a control the browser already has.

### Through a variant partial

The field gets a native-specific version carrying the bridge wiring, and the browser version stays exactly as it was:

```erb
<%# app/views/fields/_technicians.html.erb — browsers %>
<%= form.label :technician_ids, "Technicians" %>
<%= form.collection_select :technician_ids, Technician.active, :id, :name, {},
      { multiple: true, size: 8, class: "js-tomselect" } %>
```

```erb
<%# app/views/fields/_technicians.html+hotwire_native.erb — native apps %>
<div data-controller="bridge--multi-select"
     data-bridge--multi-select-title-value="Assign technicians"
     data-bridge--multi-select-empty-label-value="No technicians assigned">

  <%= form.label :technician_ids, "Technicians" %>

  <%= form.collection_select :technician_ids, Technician.active, :id, :name, {},
        {
          multiple: true,
          data: {
            "bridge--multi-select-target": "select",
            "bridge-hide-when-native": true
          }
        } %>

  <button type="button"
          class="field-summary"
          data-bridge--multi-select-target="summary"
          data-action="click->bridge--multi-select#open">
  </button>
</div>
```

And the form renders one line, unchanged, for everyone:

```erb
<%# app/views/work_orders/_form.html.erb %>
<%= render "fields/technicians", form: form %>
```

Rails picks the variant when the request came from a native app and falls back to the plain partial otherwise. The form itself never learns that any of this happened.

### What the variant buys

1. **Browsers receive no dead markup.** No hidden summary button, no `data-controller` for a component that can never load.
2. **No CSS hiding rule** is needed for elements the variant simply doesn't render.
3. **The browser's own enhancement doesn't have to ship to native.** This is the biggest one and it is easy to overlook. In the example above, browsers get `class="js-tomselect"` and native doesn't — so the native app never initializes a 60 KB widget it is about to replace. Chapter 14 is the strongest case: the web typeahead library is pure overhead in the app.
4. **Field-level reuse.** A field used on eight forms gets its native treatment once, in one partial, rather than eight times inline.

### What it costs

1. Two templates per field instead of one.
2. Caching has to be keyed by variant — see 3.7, which is where this bites.
3. **It does not replace the client-side capability check.**

That third one is the important one, and it is the reason variants and Chapter 4 are complementary rather than alternatives:

> The server knows *"this request came from a native app."* It does **not** know *"this native app has a multi-select component registered."*
>
> A user on a six-month-old build gets the variant, because their user agent still says Hotwire Native — but their app may have no `multi-select` component at all. The summary button would render with nothing behind it, and the web control would be hidden by nothing.
>
> So a variant-delivered bridge component **still** needs `this.enabled` in its controller and **still** needs the `[data-bridge-components~="multi-select"]` CSS rule. The variant answers *browser or app*; the client answers *which components this particular app actually has*. Chapter 4 is about the second question, and using variants does not make it go away.

### Choosing

| Deliver inline when | Deliver through a variant when |
|---|---|
| The wiring is data attributes on elements that already exist | Native needs elements a browser has no use for |
| The same control serves both, just nicer natively (9, 10, 12) | The browser gets a JS enhancement native shouldn't load (11, 14) |
| The field appears on one or two forms | The field appears across many forms |
| You are adding the first native-aware field to a page | The page has accumulated several and the shared template is getting noisy |

> **The component chapters all show inline delivery**, so that each one's markup and its fallback behavior sit next to each other and can be read together. Treat that as the starting point, not the destination.
>
> Expect to convert to variant partials as native-only markup accumulates; a field-partial convention such as `app/views/fields/` makes the conversion mechanical. **The bridge component itself is byte-for-byte identical either way**: same JavaScript, same Swift, same Kotlin, same contract. Only where the markup comes from changes.

## 3.6 The rule for variants

> **A variant may change presentation. It must never change data, validation, or authorization.**

Same params, same strong parameters, same model callbacks, same policy checks. The moment a native variant submits a different set of fields, you have two contracts, and "works on the web but not in the app" becomes a permanent category in your bug tracker.

The test for whether you've held the line:

**Delete every variant template and every bridge component. The application must still be completely functional for everyone — just less tailored.**

If deleting a variant breaks a feature, that feature's logic leaked into the view layer and belongs in the model or controller instead.

## 3.7 Caching — the part that bites

This is the failure mode people hit in production rather than in development, because it needs two different clients hitting the same cache.

**Fragment caching inside a variant template is safe.** `_form.html+hotwire_native.erb` has a different template digest from `_form.html.erb`, so `cache @work_order do` produces different keys in each.

**Fragment caching around a `hotwire_native_app?` conditional is not.** Same template, same digest, two possible outputs:

```erb
<%# Wrong — one cache entry, two possible renderings %>
<% cache work_order do %>
  <% if hotwire_native_app? %>…<% else %>…<% end %>
<% end %>

<%# Right — the variant is part of the key %>
<% cache [work_order, request.variant] do %>
  <% if hotwire_native_app? %>…<% else %>…<% end %>
<% end %>
```

Better still: move that conditional into a variant template and delete the branch. A conditional inside a cached fragment is a smell regardless of caching.

**HTTP caching** needs the variant in the etag, once, globally:

```ruby
# app/controllers/application_controller.rb
etag { request.variant }
```

Without it, `fresh_when` and `stale?` will happily serve a browser visitor the response generated for a native app.

> **If a CDN fronts your HTML**, it must vary on whatever distinguishes the two audiences. `Vary: User-Agent` is correct but shreds your hit rate, since every OS build is a distinct key. The usual answer is to not cache HTML at the CDN at all, and cache fragments in Rails instead, where `request.variant` is available as a key.

## 3.8 Testing both surfaces

Define the native user agents once:

```ruby
# test/test_helper.rb
HOTWIRE_NATIVE_IOS_UA = [
  "MyApp/3.4.1;",
  "Hotwire Native iOS;",
  "Turbo Native iOS;",
  "bridge-components: [form toast confirm select date-picker multi-select date-range typeahead nested-form wizard unsaved-changes];",
  "Mozilla/5.0 (iPhone; CPU iPhone OS 18_0 like Mac OS X) AppleWebKit/605.1.15"
].join(" ").freeze

HOTWIRE_NATIVE_ANDROID_UA = [
  "MyApp/3.4.1;",
  "Hotwire Native Android;",
  "Turbo Native Android;",
  "bridge-components: [form toast confirm select date-picker multi-select date-range typeahead nested-form wizard unsaved-changes];",
  "Mozilla/5.0 (Linux; Android 14) AppleWebKit/537.36 Chrome/120.0.0.0 Mobile"
].join(" ").freeze
```

Then assert on both surfaces where a variant exists:

```ruby
# test/controllers/work_orders_controller_test.rb
require "test_helper"

class WorkOrdersControllerTest < ActionDispatch::IntegrationTest
  test "browsers get the full site chrome" do
    get work_orders_path

    assert_response :success
    assert_select "header.site-header", count: 1
    assert_select "footer.site-footer", count: 1
  end

  test "native apps get the stripped layout" do
    get work_orders_path, headers: { "HTTP_USER_AGENT" => HOTWIRE_NATIVE_IOS_UA }

    assert_response :success
    assert_select "header.site-header", count: 0
    assert_select "body.native", count: 1
  end

  test "both surfaces accept the same params and persist the same record" do
    params = { work_order: { title: "Replace compressor", scheduled_on: "2026-03-14" } }

    assert_difference -> { WorkOrder.count }, 2 do
      post work_orders_path, params: params
      post work_orders_path, params: params,
           headers: { "HTTP_USER_AGENT" => HOTWIRE_NATIVE_IOS_UA }
    end

    assert_equal WorkOrder.last(2).map(&:title).uniq.size, 1
  end
end
```

That third test is the one worth copying. It is the executable form of 3.6: whatever the presentation, the two surfaces write the same record from the same params.

The discipline that follows from it:

- **Every feature gets a browser test.** That is the majority path and the source of truth.
- **Only features with a variant get a second native test**, and it asserts presentation, not persistence.
- **Bridge components get no server test at all** — they're client-side, and they're covered by the fallback system test in each component chapter.

## 3.9 Where the rest of this guide sits

Chapters 5 through 17 are the component chapters — eleven bridge components plus two that deliberately have none. The two things that vary between them are what the **server** has to do, and how the markup is **delivered**: the two questions from 3.2.

| Chapter | What the server does | Delivery as shown | In practice |
|---|---|---|---|
| 5 · Submit button | Nothing — same form everywhere | Inline | Inline |
| 6 · Flash toast | Nothing — the same flash partial | Inline | Inline |
| 7 · Confirm dialog | Nothing — the same `data-turbo-confirm` | Inline | Inline |
| 8 · Plain inputs | Nothing at all | — | — no component at all |
| 9 · Select | Nothing — the same `<select>` | Inline | Inline |
| 10 · Date picker | Nothing — the same `date_field` | Inline | Inline |
| 11 · Multi-select | Nothing — the same `<select multiple>` | Inline | **Variant** — a summary button browsers can't use |
| 12 · Date range | Nothing — the same two `date_field`s | Inline | Inline |
| 13 · Dependent selects | **A Turbo Frame endpoint** — the cascade is server-rendered | Inline, inside the frame | Same |
| 14 · Remote typeahead | **A JSON endpoint** — shared with the web typeahead | Inline | **Variant** — keeps the web typeahead library out of the app |
| 15 · Nested fieldsets | `accepts_nested_attributes_for` — plain Rails | Inline | Inline |
| 16 · Multi-step wizard | **Steps as real URLs** — the whole chapter | Inline, in the step layout | Same |
| 17 · Unsaved changes | Nothing | Inline | Inline |

Read the second column carefully. For every component through Chapter 12 it says *nothing* — the Rails app does not know those native components exist. Where it does have work to do (13, 14, 16), that work is **ordinary Rails the browser needs too**: a Turbo Frame endpoint, a JSON endpoint, steps as URLs. None of it is native-specific.

The last column is where 3.5 applies. Two rows differ from what the chapter shows, and for the same reason each time: their native path needs a summary button that browsers have no use for, and Chapter 14's browser path loads a library the app has no use for.

**The layout is the exception to the whole table, which is why it has no row in it.** It is not a component and it belongs to no chapter — it is the one genuinely native-aware thing the server does anywhere in this guide, and it is always a variant (3.4). Navigation chrome is the single piece of the site the native shell actually duplicates. Everything else stays a site, and the apps are a rendering of it.

---

# Chapter 4 — The progressive-enhancement contract

This chapter is short and it is the most important one in the guide. Skipping it produces apps that break for users on the previous app version.

## 4.1 Who is out there

At any moment, the same Rails view is being rendered for:

| Audience | Has the native component? |
|---|---|
| Desktop browser | No |
| Mobile browser | No |
| App v12 (current) | Yes |
| App v9 (user hasn't updated) | **No** — it registered fewer components |
| App on the platform you skipped | No |

You control the web deploy. You do not control which app version anyone is running. So the web must be correct on its own, and the native component is decoration that sometimes appears.

Chapter 3 handled this on the server, where the unit is the whole request and the signal is the user agent. This chapter handles it in the browser, where the unit is a single element and the signal is whether one specific component was registered. The server knows *"this is a native app."* The page knows *"this native app has a date picker."* Those are different questions and you need both.

This is why choosing variant delivery (3.5) does not let you skip this chapter. A variant partial answers the server's question — it renders native markup for native apps — but it cannot answer the page's. Row four of the table above is precisely the case it misses.

## 4.2 The three signals

**`this.enabled`** — true when the current native app registered this component. Read it before you change any DOM.

```js
connect() {
  if (!this.enabled) return   // leave the web control exactly as it was
  // … native path
}
```

**`data-bridge-components`** — the library writes the list of registered component names onto `<html>`. This is what CSS keys off:

```css
/* app/assets/stylesheets/bridge.css */
[data-bridge-components~="form"] [data-bridge-hide-when-native] {
  display: none;
}
```

The `~=` operator matches one item in a space-separated list, so `~="form"` matches `data-bridge-components="form toast date-picker"` but not `data-bridge-components="form-extras"`.

**`data-controller-optout-ios` / `-android`** — disables the component on one platform. Read back as `this.platformOptingOut`. Useful when you have shipped iOS but not Android:

```erb
<div data-controller="bridge--select" data-controller-optout-android>
```

## 4.3 The rule for every component

Every component chapter has a **Fallback** section, and every one of them satisfies this:

> Turn the native app off entirely. The page must still be fully usable, with no visual artifacts from the component that isn't there.

The practical consequence is that you **hide the web control from CSS, not from JavaScript**, and only when the native component is confirmed present. If JS hides it and then the bridge fails, the user is looking at nothing.

---

# Chapter 5 — Native submit button

*Complexity 1 · ~40 lines JS, ~70 Swift, ~80 Kotlin*

The traditional first component, and a good one, because it introduces the reply-many pattern.

## 5.1 The web problem

A long form on a phone puts its submit button below the fold. The usual fixes are all bad:

```js
// The kind of thing this replaces
document.addEventListener("focusin", () => {
  // keyboard is up; recalculate the sticky footer offset
  footer.style.bottom = `${window.innerHeight - visualViewport.height}px`
})
```

Plus sticky-footer CSS that fights iOS Safari's viewport, a disable-on-submit handler, a spinner, and a `requestSubmit` shim. Perhaps 120 lines and a class of bugs that only reproduce on real devices.

## 5.2 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `connect` | `{ submitTitle: String }` |
| Native → Web | reply to `connect` | *(none)* — fires on every tap |
| Web → Native | `submitEnabled` | `{}` |
| Web → Native | `submitDisabled` | `{}` |

Note the asymmetry: **one** `connect` message, **many** replies. Each reply is one tap of the native button.

## 5.3 Rails

```erb
<%# app/views/work_orders/_form.html.erb %>
<%= form_with(model: work_order, data: { controller: "bridge--form" }) do |form| %>
  <%= form.text_field :title %>
  <%= form.text_area :notes %>

  <%= form.submit "Save work order",
        data: {
          "bridge--form-target": "submit",
          "bridge-hide-when-native": true
        } %>
<% end %>
```

**This is the ordinary form partial — there is no variant.** It ships to desktop browsers, mobile browsers, and native apps identically. The two data attributes are inert in a browser: Stimulus finds no native bridge, so `this.enabled` is false and the controller does nothing, and the CSS rule in 5.5 never matches because `data-bridge-components` is absent. Inline delivery, in the terms of 3.2 — and the right choice here, since nothing is added to the page that a browser can't use.

## 5.4 Web

```js
// app/javascript/controllers/bridge/form_controller.js
import { BridgeComponent, BridgeElement } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "form"
  static targets = ["submit"]

  submitTargetConnected(target) {
    const submitButton = new BridgeElement(target)
    const submitTitle = submitButton.title

    this.send("connect", { submitTitle }, () => {
      submitButton.click()
    })
  }
}
```

Eleven lines. `BridgeElement#title` resolves the button's label from `data-bridge-title`, then `aria-label`, then its text content — so the native button says the same thing the web button says, in the same locale, without a second translation key.

The callback calls `.click()` on the web button, which means the native button submits the form through exactly the same code path the web button uses. Turbo, validations, `data-turbo-confirm`, everything — unchanged.

## 5.5 CSS

```css
/* app/assets/stylesheets/bridge.css */
[data-bridge-components~="form"] [data-bridge-hide-when-native] {
  display: none;
}
```

## 5.6 iOS

```swift
// ios/MyApp/Bridge/FormComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Displays a submit button in the native toolbar, which submits
/// the form on the page when tapped.
final class FormComponent: BridgeComponent {
    override class var name: String { "form" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .connect:
            handleConnectEvent(message: message)
        case .submitEnabled:
            submitBarButtonItem?.isEnabled = true
        case .submitDisabled:
            submitBarButtonItem?.isEnabled = false
        }
    }

    // MARK: Private

    private weak var submitBarButtonItem: UIBarButtonItem?

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func handleConnectEvent(message: Message) {
        guard let data: MessageData = message.data() else { return }
        configureBarButton(with: data.submitTitle)
    }

    private func configureBarButton(with title: String) {
        guard let viewController else { return }

        let action = UIAction { [unowned self] _ in
            reply(to: Event.connect.rawValue)
        }

        let item = UIBarButtonItem(title: title, primaryAction: action)
        viewController.navigationItem.rightBarButtonItem = item
        submitBarButtonItem = item
    }
}

// MARK: Events

private extension FormComponent {
    enum Event: String {
        case connect
        case submitEnabled
        case submitDisabled
    }
}

// MARK: Message data

private extension FormComponent {
    struct MessageData: Decodable {
        let submitTitle: String
    }
}
```

Three patterns here that every iOS component in this guide repeats:

- **`enum Event: String`** instead of string literals in a `switch`. Unknown events fall out of the `guard` harmlessly, and the compiler checks exhaustiveness for the ones you do handle.
- **`delegate?.destination as? UIViewController`** is how you reach the screen. It is optional, and it can be nil during teardown — always `guard`.
- **`weak var` for the UI element.** The bar button item belongs to the view controller. Holding it strongly here outlives the screen.

## 5.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/FormComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import android.view.Menu
import android.view.MenuItem
import androidx.appcompat.widget.Toolbar
import androidx.fragment.app.Fragment
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.demo.R
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Displays a submit button in the native toolbar, which submits
 * the form on the page when tapped.
 */
class FormComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private val submitButtonItemId = 37
    private var submitMenuItem: MenuItem? = null

    private val fragment: Fragment
        get() = delegate.destination.fragment

    private val toolbar: Toolbar?
        get() = fragment.view?.findViewById(R.id.toolbar)

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnectEvent(message)
            "submitEnabled" -> submitMenuItem?.isEnabled = true
            "submitDisabled" -> submitMenuItem?.isEnabled = false
            else -> Log.w("FormComponent", "Unknown event for message: $message")
        }
    }

    private fun handleConnectEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        showToolbarButton(data.title)
    }

    private fun showToolbarButton(title: String) {
        val menu = toolbar?.menu ?: return
        val order = 999 // right-most

        menu.removeItem(submitButtonItemId)
        submitMenuItem = menu.add(Menu.NONE, submitButtonItemId, order, title).apply {
            setShowAsAction(MenuItem.SHOW_AS_ACTION_ALWAYS)
            setOnMenuItemClickListener {
                replyTo("connect")
                true
            }
        }
    }

    @Serializable
    data class MessageData(
        @SerialName("submitTitle") val title: String
    )
}
```

> **`menu.removeItem` before `menu.add` is not optional.** Turbo re-renders can fire `connect` more than once on the same screen. Without the remove, you accumulate duplicate toolbar buttons. This is the single most common Android bridge bug.

## 5.8 Verify

1. Load the form in a desktop browser → the web submit button is visible and works.
2. Load it in the app → the web button is gone, a native button appears in the toolbar with the same label.
3. Tap the native button → the form submits and Turbo navigates.
4. Tap it **twice in a row** → two submissions should not both go through; wire `submitDisabled` on `turbo:submit-start`.

## 5.9 Fallback

No native component → `data-bridge-components` never contains `form` → the CSS rule doesn't match → the web button stays visible. `this.enabled` is false, so `send` is a no-op. Nothing else changes.

## 5.10 Gotchas

1. **The native button appears twice.** Android: you forgot `menu.removeItem`. iOS: you set `rightBarButtonItem` on a stale view controller — re-read `delegate?.destination` each time rather than caching it.
2. **The native button appears but does nothing.** The callback in `send` fired but `submitButton.click()` hit a `<button type="button">`. Use `type="submit"` or call `form.requestSubmit()`.
3. **Nothing appears at all.** Component name mismatch, or registration after the first `Session`. See Chapter 19.
4. **The form submits successfully but you land back on the form.** Not your component. A regression reported against iOS **1.3.0-beta** popped the form screen on *any* redirect, which cancelled the in-flight redirect and re-rendered the form from Turbo's snapshot cache. The record is created; the navigation is wrong. 20.4 entry 8 has the mechanism. Make "submit a form on the main stack and confirm you land on the record" part of your upgrade checklist.

---

# Chapter 6 — Native flash messages

*Complexity 1 · ~35 lines JS, ~90 Swift, ~25 Kotlin*

## 6.1 The web problem

Rails flashes arrive as HTML. On the web you style them; in a native app a web-rendered toast looks wrong — wrong font, wrong animation, wrong safe-area behavior, and it scrolls with the page.

The JS this replaces is a toast component with timers, a stacking manager, exit animations, and an ARIA live region.

## 6.2 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ message: String, style: String }` |

No reply. Fire and forget.

## 6.3 Rails

```erb
<%# app/views/layouts/_flash.html.erb %>
<% flash.each do |type, message| %>
  <div class="flash flash--<%= type %>"
       data-controller="bridge--toast"
       data-bridge--toast-message-value="<%= message %>"
       data-bridge--toast-style-value="<%= type %>"
       data-bridge-hide-when-native>
    <%= message %>
  </div>
<% end %>
```

## 6.4 Web

```js
// app/javascript/controllers/bridge/toast_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "toast"
  static values = {
    message: String,
    style: { type: String, default: "notice" }
  }

  connect() {
    super.connect()
    if (!this.enabled) return

    this.send("display", {
      message: this.messageValue,
      style: this.styleValue
    })
  }
}
```

> **`super.connect()` is required.** `BridgeComponent` does real work in `connect()` — it registers with the bridge and sets up restore listeners. Overriding `connect()` without calling `super` is a silent breakage: `this.enabled` and `send()` will not behave.

## 6.5 CSS

```css
/* app/assets/stylesheets/bridge.css */
[data-bridge-components~="toast"] [data-bridge-hide-when-native] {
  display: none;
}
```

## 6.6 iOS

```swift
// ios/MyApp/Bridge/ToastComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Displays a floating, self-dismissing message over the current screen.
final class ToastComponent: BridgeComponent {
    override class var name: String { "toast" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            present(text: data.message, style: Style(rawValue: data.style) ?? .notice)
        }
    }

    // MARK: Private

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func present(text: String, style: Style) {
        guard let view = viewController?.view else { return }

        let toast = ToastView(text: text, backgroundColor: style.backgroundColor)
        toast.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(toast)

        NSLayoutConstraint.activate([
            toast.leadingAnchor.constraint(
                greaterThanOrEqualTo: view.safeAreaLayoutGuide.leadingAnchor, constant: 16
            ),
            toast.trailingAnchor.constraint(
                lessThanOrEqualTo: view.safeAreaLayoutGuide.trailingAnchor, constant: -16
            ),
            toast.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            toast.bottomAnchor.constraint(
                equalTo: view.safeAreaLayoutGuide.bottomAnchor, constant: -24
            )
        ])

        toast.alpha = 0
        UIView.animate(withDuration: 0.25) {
            toast.alpha = 1
        } completion: { _ in
            UIView.animate(withDuration: 0.25, delay: 3.0) {
                toast.alpha = 0
            } completion: { _ in
                toast.removeFromSuperview()
            }
        }

        UINotificationFeedbackGenerator().notificationOccurred(style.feedback)
    }
}

// MARK: Events

private extension ToastComponent {
    enum Event: String {
        case display
    }
}

// MARK: Message data

private extension ToastComponent {
    struct MessageData: Decodable {
        let message: String
        let style: String
    }

    enum Style: String {
        case notice
        case alert

        var backgroundColor: UIColor {
            switch self {
            case .notice: .systemGreen
            case .alert: .systemRed
            }
        }

        var feedback: UINotificationFeedbackGenerator.FeedbackType {
            switch self {
            case .notice: .success
            case .alert: .error
            }
        }
    }
}
```

```swift
// ios/MyApp/Bridge/ToastView.swift
import UIKit

/// A rounded pill containing a single line of text.
final class ToastView: UIView {
    init(text: String, backgroundColor: UIColor) {
        super.init(frame: .zero)

        self.backgroundColor = backgroundColor
        layer.cornerRadius = 12
        layer.cornerCurve = .continuous

        let label = UILabel()
        label.text = text
        label.textColor = .white
        label.numberOfLines = 0
        label.font = .preferredFont(forTextStyle: .subheadline)
        label.adjustsFontForContentSizeCategory = true
        label.translatesAutoresizingMaskIntoConstraints = false

        addSubview(label)
        NSLayoutConstraint.activate([
            label.topAnchor.constraint(equalTo: topAnchor, constant: 12),
            label.bottomAnchor.constraint(equalTo: bottomAnchor, constant: -12),
            label.leadingAnchor.constraint(equalTo: leadingAnchor, constant: 16),
            label.trailingAnchor.constraint(equalTo: trailingAnchor, constant: -16)
        ])

        isAccessibilityElement = true
        accessibilityLabel = text
        accessibilityTraits = .staticText
    }

    @available(*, unavailable)
    required init?(coder: NSCoder) {
        fatalError("init(coder:) is not supported")
    }
}
```

`adjustsFontForContentSizeCategory` plus `numberOfLines = 0` is what makes this respect Dynamic Type. A web-rendered toast in a WebView will not, because the WebView does not scale with the system text size setting by default.

## 6.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/ToastComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import androidx.core.content.ContextCompat
import com.google.android.material.snackbar.Snackbar
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.demo.R
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Displays a Snackbar for a Rails flash message.
 */
class ToastComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            else -> Log.w("ToastComponent", "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val view = delegate.destination.fragment.view ?: return

        // These two colors are yours to define in res/values/colors.xml —
        // they are not provided by the library or by Material.
        val colorRes = when (data.style) {
            "alert" -> R.color.toast_alert
            else -> R.color.toast_notice
        }

        Snackbar.make(view, data.message, Snackbar.LENGTH_LONG)
            .setBackgroundTint(ContextCompat.getColor(view.context, colorRes))
            .show()
    }

    @Serializable
    data class MessageData(
        @SerialName("message") val message: String,
        @SerialName("style") val style: String
    )
}
```

Android gets this nearly free — `Snackbar` already handles queuing, swipe-to-dismiss, TalkBack announcement, and safe insets.

## 6.8 Verify

Trigger a flash (`redirect_to …, notice: "Saved"`). Browser: styled div. App: native toast, correct color, dismisses in ~3s, announced by VoiceOver/TalkBack.

## 6.9 Gotchas

1. **Two toasts on one navigation.** Turbo renders the flash partial on both the preview and the final render. Guard with `data-turbo-temporary` on the flash container, or dedupe natively by message text.
2. **iOS toast appears behind the keyboard.** It is pinned to `safeAreaLayoutGuide.bottomAnchor`, which does not track the keyboard. For forms, pin to the top instead.
3. **No toast at all on Android after a redirect — but it works on iOS and in a browser.** Not your component. After a POST, the Android client has been reported to issue the follow-up GET **twice**, once as `TURBO_STREAM` and once as `HTML`; the first consumes the flash and the second renders without it. Count the requests in the Rails log before touching the component. 20.3 entry 6.
4. **The toast shows a stale message, or none, after redirecting to a page already on the stack.** The page came from Hotwire Native's snapshot cache, not from the server, so the flash partial is whatever it was last time. 20.3 entry 5.

---

# Chapter 7 — Native confirmation dialogs

*Complexity 2 · ~30 lines JS, ~60 Swift, ~60 Kotlin*

This one is different from the others: it is not attached to a form field, it replaces a **global Turbo hook**.

## 7.1 The web problem

`data-turbo-confirm` calls `window.confirm` by default, which on a phone renders a browser-chrome dialog that says the name of your domain and looks like a security warning. So teams write a custom modal — with focus trapping, scroll locking, escape handling, and a promise-based API glued into `Turbo.setConfirmMethod`.

## 7.2 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ title: String, message: String, confirmTitle: String, cancelTitle: String }` |
| Native → Web | reply to `display` | `{ confirmed: Boolean }` |

This is the first component with a **meaningful reply**, and the first where the reply drives a JavaScript `Promise`.

## 7.3 Rails

Attach once, on the layout body:

```erb
<%# app/views/layouts/application.html.erb %>
<body data-controller="bridge--confirm">
  <%= yield %>
</body>
```

Then use `data-turbo-confirm` anywhere, unchanged:

```erb
<%= button_to "Delete work order", work_order_path(@work_order),
      method: :delete,
      data: { turbo_confirm: "This cannot be undone." } %>
```

## 7.4 Web

```js
// app/javascript/controllers/bridge/confirm_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"
import { setConfirmMethod } from "@hotwired/turbo"

export default class extends BridgeComponent {
  static component = "confirm"
  static values = {
    title: { type: String, default: "Are you sure?" },
    confirmTitle: { type: String, default: "Confirm" },
    cancelTitle: { type: String, default: "Cancel" }
  }

  connect() {
    super.connect()
    if (!this.enabled) return

    setConfirmMethod((message) => this.#confirm(message))
  }

  disconnect() {
    super.disconnect()
    setConfirmMethod((message) => Promise.resolve(window.confirm(message)))
  }

  #confirm(message) {
    return new Promise((resolve) => {
      this.send("display", {
        title: this.titleValue,
        message: message,
        confirmTitle: this.confirmTitleValue,
        cancelTitle: this.cancelTitleValue
      }, (reply) => {
        resolve(reply.data.confirmed)
      })
    })
  }
}
```

The reply resolves the promise Turbo is awaiting. Because the native side replies for **both** outcomes — confirm and cancel — the promise always settles and Turbo never hangs. This is the payoff of contract design: a component that only replied on "confirm" would leave every cancelled deletion in a permanently pending state.

## 7.5 iOS

```swift
// ios/MyApp/Bridge/ConfirmComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Replaces Turbo's confirm method with a native alert.
final class ConfirmComponent: BridgeComponent {
    override class var name: String { "confirm" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            presentAlert(with: data)
        }
    }

    // MARK: Private

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func presentAlert(with data: MessageData) {
        guard let viewController else { return }

        let alert = UIAlertController(
            title: data.title,
            message: data.message,
            preferredStyle: .alert
        )

        alert.addAction(
            UIAlertAction(title: data.cancelTitle, style: .cancel) { [unowned self] _ in
                reply(with: false)
            }
        )

        alert.addAction(
            UIAlertAction(title: data.confirmTitle, style: .destructive) { [unowned self] _ in
                reply(with: true)
            }
        )

        viewController.present(alert, animated: true)
    }

    private func reply(with confirmed: Bool) {
        reply(
            to: Event.display.rawValue,
            with: ReplyData(confirmed: confirmed)
        )
    }
}

// MARK: Events

private extension ConfirmComponent {
    enum Event: String {
        case display
    }
}

// MARK: Message data

private extension ConfirmComponent {
    struct MessageData: Decodable {
        let title: String
        let message: String
        let confirmTitle: String
        let cancelTitle: String
    }

    struct ReplyData: Encodable {
        let confirmed: Bool
    }
}
```

## 7.6 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/ConfirmComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Replaces Turbo's confirm method with a Material alert dialog.
 */
class ConfirmComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            else -> Log.w("ConfirmComponent", "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val context = delegate.destination.fragment.context ?: return

        MaterialAlertDialogBuilder(context)
            .setTitle(data.title)
            .setMessage(data.message)
            .setCancelable(true)
            .setNegativeButton(data.cancelTitle) { _, _ -> reply(false) }
            .setPositiveButton(data.confirmTitle) { _, _ -> reply(true) }
            .setOnCancelListener { reply(false) }
            .show()
    }

    private fun reply(confirmed: Boolean) {
        replyTo("display", ReplyData(confirmed))
    }

    @Serializable
    data class MessageData(
        @SerialName("title") val title: String,
        @SerialName("message") val message: String,
        @SerialName("confirmTitle") val confirmTitle: String,
        @SerialName("cancelTitle") val cancelTitle: String
    )

    @Serializable
    data class ReplyData(
        @SerialName("confirmed") val confirmed: Boolean
    )
}
```

> **`setOnCancelListener` matters.** Android users dismiss dialogs with the back gesture or an outside tap, and neither button listener fires. Without that line, roughly a third of real dismissals leave the promise pending and the app appears frozen. iOS has no equivalent problem because `UIAlertController` cannot be dismissed without choosing an action.

## 7.7 Verify

1. Browser: `data-turbo-confirm` shows `window.confirm`.
2. App: native alert with your title and button labels.
3. **Cancel by back-gesture on Android** and confirm the page still responds afterward.

## 7.8 Gotchas

1. **The dialog appears twice.** The controller is on `<body>` and something else also calls `setConfirmMethod`. Only one owner.
2. **Turbo hangs after cancel.** You did not reply on the cancel path. Reply on every path, always.
3. **`setConfirmMethod` import fails.** In some Turbo versions it is only on the `Turbo` namespace object. Use `import * as Turbo from "@hotwired/turbo"` and call `Turbo.setConfirmMethod` if the named import is unavailable. ⚠️ Version-sensitive — check against your Turbo version.
4. **The alert disappears without either action firing.** "A `UIAlertController` can't be dismissed without choosing an action" is true of the user's options and false of the app's: a deep link, a push-notification handler, or any programmatic dismissal of the presenter takes the alert away with no action. The framework's own JS-dialog handling has a reported crash from exactly this — an unreplied completion handler raising `NSInternalInconsistencyException` on the next WebView interaction (20.5 entry 11). Your component's equivalent is a permanently pending Turbo promise. If your app dismisses screens programmatically, reply from teardown as well as from the buttons.

---

# Chapter 8 — Inputs you should *not* bridge

*Complexity 1 · HTML only*

Before the harder chapters, the most valuable thing this guide can tell you: **a large fraction of "we need a native input" is solved by HTML attributes.** Bridging these costs three implementations and buys nothing.

## 8.1 Keyboards

```erb
<%= form.text_field :quantity, inputmode: "numeric", pattern: "[0-9]*" %>
<%= form.telephone_field :phone, autocomplete: "tel" %>
<%= form.email_field :email, autocomplete: "email", autocapitalize: "none" %>
<%= form.url_field :website, inputmode: "url", autocapitalize: "none" %>
<%= form.number_field :price, inputmode: "decimal" %>
```

`inputmode` controls the keyboard on both platforms. `type="number"` alone is worse — it brings a spinner on desktop and inconsistent mobile keyboards.

## 8.2 Autofill, including one-time codes

```erb
<%= form.text_field :otp,
      autocomplete: "one-time-code",
      inputmode: "numeric",
      maxlength: 6 %>
```

This gets you the iOS "From Messages" keyboard suggestion and Android SMS autofill. A bridged OTP field is strictly worse and takes 400 lines.

```erb
<%= form.text_field :street,   autocomplete: "address-line1" %>
<%= form.text_field :city,     autocomplete: "address-level2" %>
<%= form.text_field :postal,   autocomplete: "postal-code" %>
<%= form.password_field :password, autocomplete: "new-password" %>
```

Correct `autocomplete` tokens unlock the platform password manager and address autofill. This is the highest return-on-effort change in most Rails forms and it is pure HTML.

## 8.3 Keyboard navigation between fields

```erb
<%= form.text_field :first_name, enterkeyhint: "next" %>
<%= form.text_field :last_name,  enterkeyhint: "next" %>
<%= form.text_field :title,      enterkeyhint: "done" %>
```

## 8.4 The test

Bridge an input only when **at least two** of these are true:

- The native control has behavior the web genuinely cannot reproduce (haptics, system pickers, hardware access).
- The web version needs more than ~150 lines of JS to feel correct on a phone.
- The web version has a known class of mobile bugs you keep re-fixing.
- The interaction is central enough that users would describe the app as "feeling webby" without it.

One of these is not enough. Chapter 20 has the full matrix.

---

# Chapter 9 — Select → native picker

*Complexity 2 · ~65 lines JS, ~120 Swift, ~130 Kotlin*

The first component that replaces a real JavaScript library.

## 9.1 The web problem

`<select>` renders acceptably on mobile, so teams start with it — then someone needs option groups, or search, or custom option markup, and in comes Tom Select or Slim Select. Now you own:

- popover positioning that breaks inside scroll containers
- keyboard navigation and ARIA combobox roles
- a 40–70 KB dependency
- styling that has to be redone for dark mode
- a control that still doesn't feel native on either platform

Meanwhile iOS and Android both ship a genuinely good single-choice picker.

## 9.2 The design decision

The `<select>` **stays in the DOM and stays authoritative**. Native only produces an index. This keeps Chapter 1's rule intact: form serialization, validation, and Turbo all continue to read a plain `<select>`.

## 9.3 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ title: String, items: [{ title: String, index: Int, selected: Bool }] }` |
| Native → Web | reply to `display` | `{ selectedIndex: Int }` |

Cancel sends no reply. The JS side holds no lock while waiting, so that is safe here.

## 9.4 Rails

```erb
<%# app/views/work_orders/_form.html.erb %>
<div data-controller="bridge--select"
     data-bridge--select-title-value="Assign technician">
  <%= form.label :technician_id %>
  <%= form.collection_select :technician_id, Technician.active, :id, :name,
        { include_blank: "Unassigned" },
        { data: { "bridge--select-target": "select" } } %>
</div>
```

Nothing about the Rails side is bridge-specific except the target. A `select` helper, a `collection_select`, or hand-written `<option>` tags all work.

**No variant, and nothing hidden.** Unlike the submit button, the `<select>` stays visible in the native app — it is what displays the current value, and it is what gets submitted. Native only opens on top of it. So this chapter has no CSS rule at all: a browser gets a working `<select>`, and a native app gets the same `<select>` with a nicer way to change it.

## 9.5 Web

```js
// app/javascript/controllers/bridge/select_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "select"
  static targets = ["select"]
  static values = { title: { type: String, default: "Select" } }

  selectTargetConnected(select) {
    if (!this.enabled) return

    // Stop the browser's own picker from opening; native handles the tap.
    select.addEventListener("mousedown", this.#openNativePicker)
    select.setAttribute("aria-haspopup", "listbox")
  }

  selectTargetDisconnected(select) {
    select.removeEventListener("mousedown", this.#openNativePicker)
  }

  #openNativePicker = (event) => {
    event.preventDefault()
    event.target.blur()

    const select = this.selectTarget
    const items = Array.from(select.options).map((option, index) => ({
      title: option.text,
      index: index,
      selected: index === select.selectedIndex
    }))

    this.send("display", { title: this.titleValue, items }, (reply) => {
      this.#applySelection(reply.data.selectedIndex)
    })
  }

  #applySelection(selectedIndex) {
    const select = this.selectTarget
    if (selectedIndex == null || selectedIndex === select.selectedIndex) return

    select.selectedIndex = selectedIndex
    select.dispatchEvent(new Event("input", { bubbles: true }))
    select.dispatchEvent(new Event("change", { bubbles: true }))
  }
}
```

Two lines do most of the work, and both are easy to omit:

- **`event.preventDefault()` on `mousedown`** stops the native browser picker from also opening. Without it, the user gets two pickers stacked.
- **Dispatching `input` *and* `change`** is what makes the rest of your app notice. Turbo's auto-submit, other Stimulus controllers, and client-side validation all listen for one or the other. Setting `.selectedIndex` in code fires neither on its own.

## 9.6 iOS

```swift
// ios/MyApp/Bridge/SelectComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Presents a native action sheet for a web <select> element and
/// replies with the index of the chosen option.
final class SelectComponent: BridgeComponent {
    override class var name: String { "select" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            presentSheet(title: data.title, items: data.items)
        }
    }

    // MARK: Private

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func presentSheet(title: String, items: [Item]) {
        guard let viewController else { return }

        let alert = UIAlertController(
            title: title,
            message: nil,
            preferredStyle: .actionSheet
        )

        for item in items {
            let action = UIAlertAction(title: item.title, style: .default) { [unowned self] _ in
                reply(
                    to: Event.display.rawValue,
                    with: SelectionData(selectedIndex: item.index)
                )
            }
            action.setValue(item.selected, forKey: "checked")
            alert.addAction(action)
        }

        alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))

        // Required on iPad, where action sheets must originate from a rect.
        if let popover = alert.popoverPresentationController,
           let sourceView = viewController.view {
            popover.sourceView = sourceView
            popover.sourceRect = CGRect(
                x: sourceView.bounds.midX,
                y: sourceView.bounds.midY,
                width: 0,
                height: 0
            )
            popover.permittedArrowDirections = []
        }

        viewController.present(alert, animated: true)
    }
}

// MARK: Events

private extension SelectComponent {
    enum Event: String {
        case display
    }
}

// MARK: Message data

private extension SelectComponent {
    struct MessageData: Decodable {
        let title: String
        let items: [Item]
    }

    struct Item: Decodable {
        let title: String
        let index: Int
        let selected: Bool
    }

    struct SelectionData: Encodable {
        let selectedIndex: Int
    }
}
```

> ⚠️ **`action.setValue(_:forKey: "checked")` is undocumented KVC.** It draws the checkmark on the selected row and is widely used, but it is not public API and could break in a future iOS release. If you want to stay strictly within public API, drop that line and prefix the selected item's title with a checkmark character instead, or use the `UITableView` variant described in 9.10.

> **`popoverPresentationController` is not optional on iPad.** An action sheet presented without a source rect crashes on iPad. The upstream demo reads the source rect from the web element's bounding box and passes it through the message — a refinement worth adopting once the basic version works.

## 9.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/SelectComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Presents a native single-choice dialog for a web <select> element
 * and replies with the index of the chosen option.
 */
class SelectComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            else -> Log.w("SelectComponent", "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val context = delegate.destination.fragment.context ?: return

        val titles = data.items.map { it.title }.toTypedArray()
        val checkedIndex = data.items.indexOfFirst { it.selected }

        MaterialAlertDialogBuilder(context)
            .setTitle(data.title)
            .setSingleChoiceItems(titles, checkedIndex) { dialog, which ->
                dialog.dismiss()
                replyTo("display", SelectionData(data.items[which].index))
            }
            .setNegativeButton(android.R.string.cancel, null)
            .show()
    }

    @Serializable
    data class MessageData(
        @SerialName("title") val title: String,
        @SerialName("items") val items: List<Item>
    )

    @Serializable
    data class Item(
        @SerialName("title") val title: String,
        @SerialName("index") val index: Int,
        @SerialName("selected") val selected: Boolean
    )

    @Serializable
    data class SelectionData(
        @SerialName("selectedIndex") val selectedIndex: Int
    )
}
```

`setSingleChoiceItems` gives you radio buttons, the current selection pre-checked, and correct TalkBack semantics for free. It scrolls, so long option lists work without extra effort.

## 9.8 Verify

1. Browser: the `<select>` opens the browser's own picker. Unchanged.
2. App: tapping opens the native sheet with the current option checked.
3. Choose a different option → the web `<select>` visibly updates.
4. Submit → the server receives the newly chosen value.
5. Cancel → nothing changes and the page still responds.

Step 4 is the one people skip and the one that catches a missing `change` dispatch.

## 9.9 Fallback

`this.enabled` is false → no `mousedown` listener is attached → the browser's picker opens normally. The `<select>` was never hidden, so there is nothing to restore.

## 9.10 Variations

- **Long lists (>20 options)** — swap the action sheet for a pushed `UITableViewController` with a `UISearchController`, and the Material dialog for a full-screen `DialogFragment`. The contract does not change; only the native presentation does.
- **Option groups** — add `group: String?` to `Item` and render section headers. iOS action sheets can't do sections, so this pushes you to the table-view variant.
- **Multi-select** — a different component, not a flag on this one: the reply becomes `{ selectedIndexes: [Int] }` and the sheet needs a Done button. That is Chapter 11.

## 9.11 Gotchas

1. **Two pickers open at once.** Missing `event.preventDefault()`, or you bound `click` instead of `mousedown` — the browser opens its picker on mousedown, before click.
2. **Selection doesn't persist after submit.** Missing `change` dispatch, or you set `option.selected` on the wrong option. Set `select.selectedIndex`.
3. **Crash on iPad.** Missing `popoverPresentationController` configuration.
4. **The dialog reopens immediately after choosing.** You called `blur()` after `preventDefault()` but the select still had focus when Turbo restored the page. Confirm `selectTargetDisconnected` removes the listener.

> ⚠️ **The least certain assumption in this chapter.** It is not an API contract but browser behavior, which varies by engine and has changed before. The design depends on `preventDefault()` during `mousedown` reliably suppressing **the WebView's own** `<select>` picker — on WKWebView and on Android's Chromium WebView, across the OS versions you support.
>
> Touch event ordering on a `<select>` is browser-specific and has changed before. If it fails, the symptom is unmistakable: **two pickers, the system one on top of yours.** Try `pointerdown` first, since it fires ahead of `mousedown` in both engines.
>
> If neither suppresses it, stop intercepting and switch to Chapter 11's shape instead — hide the `<select>` with CSS and give native its own summary button to tap. That pattern needs no interception at all, which is why Chapter 11 does not carry this warning. Test this on a real device before building more components on top of the interception approach; Chapters 10 and 12 make the same assumption about `<input type="date">`.

---

# Chapter 10 — Date picker

*Complexity 2 · ~75 lines JS, ~170 Swift, ~90 Kotlin*

The flagship of the tractable components: high user-visible payoff, a real JS library removed, and one genuinely subtle correctness problem.

## 10.1 The web problem

`<input type="date">` is inconsistent enough across browsers that most production Rails apps reach for flatpickr or air-datepicker. That brings:

- 40–60 KB of JS and CSS
- locale configuration duplicated from your Rails locale
- min/max enforcement reimplemented in JS
- a calendar popover that fights mobile viewports and virtual keyboards
- **and the timezone bug** — the one where a user picks March 14th, the picker produces a `Date` at local midnight, something serializes it as UTC, and the server stores March 13th

That last item is the real cost. It is not a styling problem, it is a correctness problem, and it is the reason this chapter is stricter than the others.

## 10.2 The non-negotiable rule

> **A calendar date is a string, not a moment in time.** It crosses the bridge as `"2026-03-14"` and never as a timestamp, an epoch offset, or a `Date` object.

Both native pickers want to work in `Date`/epoch-millis terms. So both implementations below convert at the boundary, using a **fixed UTC calendar**, and never touch the device timezone. The device being in Auckland or Los Angeles must not change which string is produced.

## 10.3 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ title: String, value: String?, min: String?, max: String? }` |
| Native → Web | reply to `display` | `{ value: String }` |

All four date fields are `YYYY-MM-DD`. `value` is `null` when the field is empty. Cancel produces no reply.

```
Web                                          Native
 │                                             │
 │ user taps the date field                    │
 │                                             │
 │ send("display", {                           │
 │   title: "Scheduled date",                  │
 │   value: "2026-03-14",                      │
 │   min:   "2026-03-12",                      │
 │   max:   "2027-03-12" })                    │
 ├────────────────────────────────────────────▶│ parse as UTC
 │                                             │ present UIDatePicker /
 │                                             │   MaterialDatePicker
 │                                             │
 │                                  ┌──────────┤ user taps Done
 │ callback(reply)                  │          │
 │◀─────────────────────────────────┘          │ format as UTC
 │ input.value = reply.data.value              │ replyTo("display",
 │ dispatch input + change                     │   { value: "2026-03-19" })
 │                                             │
 │ user taps Cancel  ──────────────────────────│ (no reply — JS holds no lock)
```

## 10.4 Rails

```erb
<%# app/views/work_orders/_form.html.erb %>
<div data-controller="bridge--date-picker"
     data-bridge--date-picker-title-value="Scheduled date">
  <%= form.label :scheduled_on %>
  <%= form.date_field :scheduled_on,
        min: Date.current,
        max: 1.year.from_now.to_date,
        data: { "bridge--date-picker-target": "input" } %>
</div>
```

`form.date_field` already emits `value`, `min`, and `max` as `YYYY-MM-DD`, which is exactly the wire format. No serialization code is needed on the Rails side at all — and no variant, because the same `date_field` serves a browser perfectly well.

This is the clearest illustration of the architecture in the whole guide: the native picker's entire job is to write a string into an `<input type="date">` that Rails already rendered, already constrained, and already knows how to parse. Delete the iOS and Android code and the field still works everywhere.

Keep the bounds on the server too. The picker constrains the UI; the model constrains the truth — the `min` and `max` attributes are a convenience for the client, and the validation is the actual rule:

```ruby
# app/models/work_order.rb
class WorkOrder < ApplicationRecord
  validates :scheduled_on, presence: true
  validate :scheduled_on_within_window

  private

  def scheduled_on_within_window
    return if scheduled_on.blank?

    if scheduled_on < Date.current
      errors.add(:scheduled_on, "can't be in the past")
    elsif scheduled_on > 1.year.from_now.to_date
      errors.add(:scheduled_on, "can't be more than a year out")
    end
  end
end
```

## 10.5 Web

```js
// app/javascript/controllers/bridge/date_picker_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "date-picker"
  static targets = ["input"]
  static values = { title: { type: String, default: "Select a date" } }

  inputTargetConnected(input) {
    if (!this.enabled) return

    // Prevent the browser's own date picker and keyboard.
    input.readOnly = true
    input.addEventListener("mousedown", this.#openNativePicker)
    input.setAttribute("aria-haspopup", "dialog")
  }

  inputTargetDisconnected(input) {
    input.readOnly = false
    input.removeEventListener("mousedown", this.#openNativePicker)
  }

  #openNativePicker = (event) => {
    event.preventDefault()

    const input = this.inputTarget

    this.send("display", {
      title: this.titleValue,
      value: input.value || null,
      min: input.min || null,
      max: input.max || null
    }, (reply) => {
      this.#applyValue(reply.data.value)
    })
  }

  #applyValue(value) {
    const input = this.inputTarget
    if (!value || value === input.value) return

    input.value = value
    input.dispatchEvent(new Event("input", { bubbles: true }))
    input.dispatchEvent(new Event("change", { bubbles: true }))
  }
}
```

`input.readOnly = true` rather than `disabled` — a disabled input is not submitted with the form, which would silently drop the field. Read-only inputs submit normally, don't raise the keyboard, and still get focus styling.

> ⚠️ **Two related assumptions to test on a device, not to take on trust.** First, that `readOnly` suppresses the WebView's own date picker: the HTML spec applies `readonly` to `date` inputs, but WebKit and Chromium have historically differed on whether a read-only date field still opens its calendar on tap. Second, the `mousedown` interception from 9.5, with the same caveat spelled out there.
>
> If either fails you get two pickers. The fallback is the same: hide the input with CSS and give native a summary button to tap, as Chapter 11 does.

The value read back from `input.value` on a `date` input is always `YYYY-MM-DD` regardless of how the browser displays it. That is the property the whole design rests on.

## 10.6 iOS

```swift
// ios/MyApp/Bridge/DatePickerComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Presents a native date picker for a web <input type="date"> and
/// replies with the selected date as an ISO-8601 calendar date.
final class DatePickerComponent: BridgeComponent {
    override class var name: String { "date-picker" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            presentPicker(with: data)
        }
    }

    // MARK: Private

    /// A fixed, timezone-independent formatter. The device's locale and
    /// timezone must never influence the string that crosses the bridge.
    private static let formatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.calendar = Calendar(identifier: .gregorian)
        formatter.locale = Locale(identifier: "en_US_POSIX")
        formatter.timeZone = TimeZone(identifier: "UTC")
        formatter.dateFormat = "yyyy-MM-dd"
        return formatter
    }()

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func presentPicker(with data: MessageData) {
        guard let viewController else { return }

        let picker = UIDatePicker()
        picker.datePickerMode = .date
        picker.preferredDatePickerStyle = .inline
        picker.calendar = Calendar(identifier: .gregorian)
        picker.timeZone = TimeZone(identifier: "UTC")

        if let value = data.value, let date = Self.formatter.date(from: value) {
            picker.date = date
        }
        if let min = data.min, let date = Self.formatter.date(from: min) {
            picker.minimumDate = date
        }
        if let max = data.max, let date = Self.formatter.date(from: max) {
            picker.maximumDate = date
        }

        let sheet = DatePickerSheetViewController(
            picker: picker,
            sheetTitle: data.title
        ) { [unowned self] selectedDate in
            reply(
                to: Event.display.rawValue,
                with: SelectionData(value: Self.formatter.string(from: selectedDate))
            )
        }

        let navigationController = UINavigationController(rootViewController: sheet)
        if let presentation = navigationController.sheetPresentationController {
            presentation.detents = [.medium(), .large()]
            presentation.prefersGrabberVisible = true
        }

        viewController.present(navigationController, animated: true)
    }
}

// MARK: Events

private extension DatePickerComponent {
    enum Event: String {
        case display
    }
}

// MARK: Message data

private extension DatePickerComponent {
    struct MessageData: Decodable {
        let title: String
        let value: String?
        let min: String?
        let max: String?
    }

    struct SelectionData: Encodable {
        let value: String
    }
}
```

```swift
// ios/MyApp/Bridge/DatePickerSheetViewController.swift
import UIKit

/// A modal sheet wrapping a UIDatePicker, with Cancel and Done.
final class DatePickerSheetViewController: UIViewController {
    private let picker: UIDatePicker
    private let onDone: (Date) -> Void

    init(picker: UIDatePicker, sheetTitle: String, onDone: @escaping (Date) -> Void) {
        self.picker = picker
        self.onDone = onDone
        super.init(nibName: nil, bundle: nil)
        title = sheetTitle
    }

    @available(*, unavailable)
    required init?(coder: NSCoder) {
        fatalError("init(coder:) is not supported")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        view.backgroundColor = .systemBackground

        navigationItem.leftBarButtonItem = UIBarButtonItem(
            systemItem: .cancel,
            primaryAction: UIAction { [unowned self] _ in
                dismiss(animated: true)
            }
        )

        navigationItem.rightBarButtonItem = UIBarButtonItem(
            systemItem: .done,
            primaryAction: UIAction { [unowned self] _ in
                onDone(picker.date)
                dismiss(animated: true)
            }
        )

        picker.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(picker)

        NSLayoutConstraint.activate([
            picker.topAnchor.constraint(
                equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 8
            ),
            picker.leadingAnchor.constraint(
                equalTo: view.leadingAnchor, constant: 16
            ),
            picker.trailingAnchor.constraint(
                equalTo: view.trailingAnchor, constant: -16
            )
        ])
    }
}
```

> **Why `picker.timeZone = TimeZone(identifier: "UTC")` and a matching formatter.** `UIDatePicker` produces a `Date`, which is an instant, not a calendar date. Set the picker's timezone to UTC and format with a UTC formatter and the round-trip is exact. Omit either and a user in UTC+13 who picks the 14th sends the 13th. This is the bug from 10.1, reproduced natively.

## 10.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/DatePickerComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import com.google.android.material.datepicker.CalendarConstraints
import com.google.android.material.datepicker.CompositeDateValidator
import com.google.android.material.datepicker.DateValidatorPointBackward
import com.google.android.material.datepicker.DateValidatorPointForward
import com.google.android.material.datepicker.MaterialDatePicker
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneOffset

/**
 * Presents a Material date picker for a web <input type="date"> and
 * replies with the selected date as an ISO-8601 calendar date.
 */
class DatePickerComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private companion object {
        const val TAG = "DatePickerComponent"
        const val MILLIS_PER_DAY = 86_400_000L
    }

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            else -> Log.w(TAG, "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val fragment = delegate.destination.fragment

        val builder = MaterialDatePicker.Builder.datePicker()
            .setTitleText(data.title)
            .setInputMode(MaterialDatePicker.INPUT_MODE_CALENDAR)

        toUtcMillis(data.value)?.let { builder.setSelection(it) }

        buildConstraints(data)?.let { builder.setCalendarConstraints(it) }

        val picker = builder.build()

        picker.addOnPositiveButtonClickListener { selectionMillis ->
            replyTo("display", SelectionData(toIsoDate(selectionMillis)))
        }

        picker.show(fragment.childFragmentManager, TAG)
    }

    private fun buildConstraints(data: MessageData): CalendarConstraints? {
        val validators = buildList {
            toUtcMillis(data.min)?.let { add(DateValidatorPointForward.from(it)) }
            // `before` is exclusive, so shift by one day to make `max` inclusive.
            toUtcMillis(data.max)?.let {
                add(DateValidatorPointBackward.before(it + MILLIS_PER_DAY))
            }
        }

        if (validators.isEmpty()) return null

        return CalendarConstraints.Builder()
            .setValidator(CompositeDateValidator.allOf(validators))
            .build()
    }

    /** "2026-03-14" -> epoch millis at UTC midnight. */
    private fun toUtcMillis(value: String?): Long? {
        if (value.isNullOrBlank()) return null
        return runCatching {
            LocalDate.parse(value).atStartOfDay(ZoneOffset.UTC).toInstant().toEpochMilli()
        }.onFailure { Log.w(TAG, "Unparseable date: $value") }.getOrNull()
    }

    /** Epoch millis -> "2026-03-14", interpreted at UTC. */
    private fun toIsoDate(millis: Long): String =
        Instant.ofEpochMilli(millis).atZone(ZoneOffset.UTC).toLocalDate().toString()

    @Serializable
    data class MessageData(
        @SerialName("title") val title: String,
        @SerialName("value") val value: String? = null,
        @SerialName("min") val min: String? = null,
        @SerialName("max") val max: String? = null
    )

    @Serializable
    data class SelectionData(
        @SerialName("value") val value: String
    )
}
```

Two Android-specific notes worth internalizing:

- **`MaterialDatePicker` already works in UTC.** Its selection value is documented as UTC epoch milliseconds, which is why `ZoneOffset.UTC` appears on both conversions and `ZoneId.systemDefault()` appears nowhere. Using the system zone here is the most common way to reintroduce the off-by-one.
- **`DateValidatorPointBackward.before()` is exclusive.** Passing `max` directly makes the maximum date itself unselectable. The `+ MILLIS_PER_DAY` shift is what makes `max` inclusive and consistent with the HTML `max` attribute.

> ⚠️ **`MaterialDatePicker` is a `DialogFragment`.** On configuration change (rotation, dark-mode toggle, font-size change) the fragment is recreated and the listener added with `addOnPositiveButtonClickListener` is **lost** — the user picks a date and nothing happens. The robust fix is to re-attach the listener in the fragment's `onResume` by looking the picker up by tag in `childFragmentManager`. Verify this against your target library version before shipping; it is the most likely thing in this chapter to bite in production.

## 10.8 Verify

Functional:

1. Browser: the field behaves as a normal `<input type="date">`.
2. App: tapping opens the native picker, pre-set to the current value, with dates outside min/max unselectable.
3. Choose a date → the field updates and shows the same date.
4. Submit → the server stores the same date you chose.
5. Cancel → the field is unchanged.

Timezone — the test that actually matters:

6. Set the **device** to `Pacific/Auckland` (UTC+13). Pick the 14th. Submit. The server must store the 14th.
7. Set the device to `Pacific/Honolulu` (UTC−10). Pick the 14th. Submit. The server must store the 14th.

Run steps 6 and 7 on both platforms before believing the component works. Everything else in this chapter can be correct while those two still fail.

As a regression test on the Rails side:

```ruby
# test/system/work_orders_test.rb
require "application_system_test_case"

class WorkOrdersTest < ApplicationSystemTestCase
  test "the date field works without the native bridge" do
    visit new_work_order_path

    fill_in "Title", with: "Replace compressor"
    fill_in "Scheduled on", with: "2026-03-14"
    click_on "Save work order"

    assert_text "Work order was successfully created"
    assert_equal Date.new(2026, 3, 14), WorkOrder.last.scheduled_on
  end
end
```

That test never runs the bridge — and that is the point. It pins the fallback path, which is the path most of your users are on.

## 10.9 Fallback

`this.enabled` is false → the input is never made read-only, no listener is attached → the browser's own date input behaves normally. Because Rails emitted `value`, `min`, and `max` as plain HTML attributes, constraint enforcement works in the browser too.

## 10.10 Variations

- **Time and date-time** — `datePickerMode = .time` / `MaterialTimePicker`, wire format `HH:mm`. Watch `is24HourFormat` on Android so the picker matches the device's clock preference.
- **Date ranges** — Android has `MaterialDatePicker.Builder.dateRangePicker()`; iOS has no first-party equivalent and needs a two-field sheet. Genuinely asymmetric; that is Chapter 12.
- **Relative shortcuts** ("Today", "Next Monday") — add them as bar buttons on the sheet. They produce the same reply, so no contract change.

## 10.11 Gotchas

1. **Off by one day.** Some conversion used the device timezone. Audit for `ZoneId.systemDefault()` on Android and any `DateFormatter` without an explicit `timeZone` on iOS.
2. **The max date can't be selected on Android.** `DateValidatorPointBackward.before()` is exclusive — add a day.
3. **The keyboard appears behind the picker on iOS.** The input still had focus. `readOnly = true` normally prevents this; if it persists, call `input.blur()` in the mousedown handler.
4. **Picking a date does nothing after rotating the device on Android.** The `DialogFragment` listener was lost. See the warning in 10.7.
5. **The field submits blank.** You used `disabled` instead of `readOnly`.

---

# Chapter 11 — Multi-select with search

*Complexity 3 · ~80 lines JS, ~210 Swift, ~70 Kotlin*

The first component where the native version is a **screen**, not a sheet — and the first that introduces an inverse CSS rule: an element that exists only for native.

## 11.1 The web problem

`<select multiple>` is genuinely bad on every platform, so nobody ships it. Teams reach for Tom Select in multi mode, or Select2, or hand-roll a checkbox list in a popover. Whichever it is, you now own:

- chip rendering, overflow, and "+3 more" truncation
- async option loading with a spinner state
- select-all / clear-all semantics
- keyboard navigation across a two-dimensional widget
- a search box with its own debounce and empty state
- 40–70 KB of JS, plus a stylesheet to re-theme for dark mode

And on a phone it is still a popover full of small tap targets.

Both platforms have a much better answer for "choose several things from a list": a full screen with a search field and checkmarks. It is the pattern the OS uses for its own settings.

## 11.2 The design decision

Same as Chapter 9: **the `<select multiple>` stays in the DOM and stays authoritative.** Native produces a list of indexes and nothing else.

But unlike Chapter 9, the web control is not usable as the native trigger — a `<select multiple>` renders as a tall listbox, and you do not want it on screen in the app. So this chapter needs two elements:

| Element | Browser | Native app |
|---|---|---|
| `<select multiple>` | visible (or enhanced by whatever you already use) | hidden by CSS |
| summary button | hidden by CSS | visible, opens the native screen |

That second row is new. Chapter 5 hid a web element under native; here we also **reveal** an element that only makes sense under native.

The version below does both with one template and two CSS rules, which keeps the chapter to a single Rails snippet. But a summary button is exactly the case 3.5 describes: a real DOM node that every browser receives and can never use. **In production this field is a good candidate for a variant partial** — `app/views/fields/_technicians.html+hotwire_native.erb` — so browsers get the plain `<select multiple>` and nothing else. 3.5 shows that version side by side with this one. The bridge component, its contract, and the Swift and Kotlin below are identical either way.

## 11.3 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ title: String, searchPlaceholder: String, items: [{ title: String, index: Int, selected: Bool }] }` |
| Native → Web | reply to `display` | `{ selectedIndexes: [Int] }` |

Cancel sends no reply. Done always replies — including with an empty array, which is how "deselect everything" is expressed. That distinction matters: `[]` means the user cleared the field; no reply means they backed out.

## 11.4 Rails

```erb
<%# app/views/work_orders/_form.html.erb %>
<div data-controller="bridge--multi-select"
     data-bridge--multi-select-title-value="Assign technicians"
     data-bridge--multi-select-search-placeholder-value="Search technicians"
     data-bridge--multi-select-empty-label-value="No technicians assigned">

  <%= form.label :technician_ids, "Technicians" %>

  <%= form.collection_select :technician_ids, Technician.active, :id, :name,
        {},
        {
          multiple: true,
          size: 8,
          data: {
            "bridge--multi-select-target": "select",
            "bridge-hide-when-native": true
          }
        } %>

  <button type="button"
          class="bridge-native-only field-summary"
          data-bridge--multi-select-target="summary"
          data-action="click->bridge--multi-select#open">
  </button>
</div>
```

Two Rails details worth knowing:

- `collection_select` with `multiple: true` emits a **hidden field with an empty value** before the select. That is what lets the form submit "nothing selected" rather than omitting the key entirely. Do not remove it; the native path depends on it exactly as the web path does.
- The `<button>` is `type="button"`. Inside a form, a button with no type is `type="submit"`, and a summary button that submits the form is a memorable bug.

## 11.5 Web

```js
// app/javascript/controllers/bridge/multi_select_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "multi-select"
  static targets = ["select", "summary"]
  static values = {
    title: { type: String, default: "Select" },
    searchPlaceholder: { type: String, default: "Search" },
    emptyLabel: { type: String, default: "None selected" }
  }

  connect() {
    super.connect()
    if (!this.enabled) return

    this.#renderSummary()
  }

  open() {
    if (!this.enabled) return

    const items = Array.from(this.selectTarget.options).map((option, index) => ({
      title: option.text,
      index: index,
      selected: option.selected
    }))

    this.send("display", {
      title: this.titleValue,
      searchPlaceholder: this.searchPlaceholderValue,
      items: items
    }, (reply) => {
      this.#applySelection(reply.data.selectedIndexes)
    })
  }

  #applySelection(indexes) {
    // The element can be gone by the time the user finishes choosing —
    // a Turbo Frame may have replaced it while the native screen was open.
    if (!this.element.isConnected || !Array.isArray(indexes)) return

    const wanted = new Set(indexes)
    const select = this.selectTarget

    Array.from(select.options).forEach((option, index) => {
      option.selected = wanted.has(index)
    })

    select.dispatchEvent(new Event("input", { bubbles: true }))
    select.dispatchEvent(new Event("change", { bubbles: true }))

    this.#renderSummary()
  }

  #renderSummary() {
    if (!this.hasSummaryTarget) return

    const labels = Array.from(this.selectTarget.selectedOptions).map((o) => o.text)
    this.summaryTarget.textContent = labels.length
      ? labels.join(", ")
      : this.emptyLabelValue
  }
}
```

The `isConnected` guard is the first appearance of a problem that gets worse in later chapters: **a native screen can outlive the DOM that opened it.** The user taps the summary, the native list pushes, and while it is open a Turbo Stream or frame replaces the form. When they tap Done, the callback still fires — against an element that is no longer in the document. Writing to it is harmless but pointless; reading from it to build a summary is how you get a `TypeError` in production. Guard it.

## 11.6 CSS

```css
/* app/assets/stylesheets/bridge.css */

/* Hide the web control once native can replace it. */
[data-bridge-components~="multi-select"] [data-bridge-hide-when-native] {
  display: none;
}

/* The inverse: elements that exist only for native apps. */
.bridge-native-only {
  display: none;
}

[data-bridge-components~="multi-select"] .bridge-native-only {
  display: block;
}
```

`.bridge-native-only` is scoped to the component that needs it rather than being global, so a page that shows the multi-select but not, say, the typeahead doesn't reveal the wrong element. Prefer one rule per component over a single global `[data-bridge-components] .bridge-native-only`. Appendix A.6 collects every rule the guide adds, in one file.

## 11.7 iOS

```swift
// ios/MyApp/Bridge/MultiSelectComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Presents a searchable, multiple-choice list for a web <select multiple>
/// and replies with the indexes of the chosen options.
final class MultiSelectComponent: BridgeComponent {
    override class var name: String { "multi-select" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            presentList(with: data)
        }
    }

    // MARK: Private

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func presentList(with data: MessageData) {
        guard let viewController else { return }

        let items = data.items.map { MultiSelectItem(title: $0.title, index: $0.index) }
        let selected = Set(data.items.filter(\.selected).map(\.index))

        let list = MultiSelectListViewController(
            items: items,
            selected: selected,
            listTitle: data.title,
            searchPlaceholder: data.searchPlaceholder
        ) { [unowned self] chosen in
            reply(
                to: Event.display.rawValue,
                with: SelectionData(selectedIndexes: chosen.sorted())
            )
        }

        let navigationController = UINavigationController(rootViewController: list)
        navigationController.modalPresentationStyle = .formSheet
        viewController.present(navigationController, animated: true)
    }
}

// MARK: Events

private extension MultiSelectComponent {
    enum Event: String {
        case display
    }
}

// MARK: Message data

private extension MultiSelectComponent {
    struct MessageData: Decodable {
        let title: String
        let searchPlaceholder: String
        let items: [Item]
    }

    struct Item: Decodable {
        let title: String
        let index: Int
        let selected: Bool
    }

    struct SelectionData: Encodable {
        let selectedIndexes: [Int]
    }
}
```

```swift
// ios/MyApp/Bridge/MultiSelectListViewController.swift
import UIKit

struct MultiSelectItem {
    let title: String
    let index: Int
}

/// A searchable list with checkmarks, Cancel and Done.
final class MultiSelectListViewController: UITableViewController, UISearchResultsUpdating {
    private let items: [MultiSelectItem]
    private let searchPlaceholder: String
    private let onDone: (Set<Int>) -> Void

    private var visible: [MultiSelectItem]
    private var selected: Set<Int>

    init(items: [MultiSelectItem],
         selected: Set<Int>,
         listTitle: String,
         searchPlaceholder: String,
         onDone: @escaping (Set<Int>) -> Void) {
        self.items = items
        self.visible = items
        self.selected = selected
        self.searchPlaceholder = searchPlaceholder
        self.onDone = onDone

        super.init(style: .insetGrouped)
        title = listTitle
    }

    @available(*, unavailable)
    required init?(coder: NSCoder) {
        fatalError("init(coder:) is not supported")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "item")

        let search = UISearchController(searchResultsController: nil)
        search.searchResultsUpdater = self
        search.obscuresBackgroundDuringPresentation = false
        search.searchBar.placeholder = searchPlaceholder
        navigationItem.searchController = search
        navigationItem.hidesSearchBarWhenScrolling = false

        navigationItem.leftBarButtonItem = UIBarButtonItem(
            systemItem: .cancel,
            primaryAction: UIAction { [unowned self] _ in
                dismiss(animated: true)
            }
        )

        navigationItem.rightBarButtonItem = UIBarButtonItem(
            systemItem: .done,
            primaryAction: UIAction { [unowned self] _ in
                onDone(selected)
                dismiss(animated: true)
            }
        )

        updatePrompt()
    }

    // MARK: Table

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        visible.count
    }

    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "item", for: indexPath)
        let item = visible[indexPath.row]

        var configuration = cell.defaultContentConfiguration()
        configuration.text = item.title
        cell.contentConfiguration = configuration
        cell.accessoryType = selected.contains(item.index) ? .checkmark : .none
        cell.accessibilityTraits = selected.contains(item.index) ? [.button, .selected] : [.button]

        return cell
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let item = visible[indexPath.row]

        if selected.contains(item.index) {
            selected.remove(item.index)
        } else {
            selected.insert(item.index)
        }

        tableView.deselectRow(at: indexPath, animated: true)
        tableView.reloadRows(at: [indexPath], with: .none)
        updatePrompt()
    }

    // MARK: Search

    func updateSearchResults(for searchController: UISearchController) {
        let query = searchController.searchBar.text?
            .trimmingCharacters(in: .whitespacesAndNewlines) ?? ""

        visible = query.isEmpty
            ? items
            : items.filter { $0.title.localizedCaseInsensitiveContains(query) }

        tableView.reloadData()
    }

    // MARK: Private

    private func updatePrompt() {
        navigationItem.prompt = selected.isEmpty
            ? nil
            : String(localized: "\(selected.count) selected")
    }
}
```

Two things this gets right that a hand-rolled web version usually doesn't:

- **Search filters the display, not the selection.** `selected` holds source indexes, and `visible` is only what's on screen. Selecting something, searching for something else, and selecting that too keeps both — which is what users expect and what most web multi-selects get wrong.
- **`accessibilityTraits` includes `.selected`.** Without it VoiceOver reads the row label and nothing about whether it is checked, because `accessoryType` is visual only.

## 11.8 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/MultiSelectComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Presents a multiple-choice dialog for a web <select multiple>
 * and replies with the indexes of the chosen options.
 */
class MultiSelectComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            else -> Log.w("MultiSelectComponent", "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val context = delegate.destination.fragment.context ?: return

        val titles = data.items.map { it.title }.toTypedArray()
        val checked = BooleanArray(data.items.size) { data.items[it].selected }

        MaterialAlertDialogBuilder(context)
            .setTitle(data.title)
            .setMultiChoiceItems(titles, checked) { _, which, isChecked ->
                checked[which] = isChecked
            }
            .setNeutralButton("Clear all") { _, _ ->
                replyTo("display", SelectionData(emptyList()))
            }
            .setNegativeButton(android.R.string.cancel, null)
            .setPositiveButton(android.R.string.ok) { _, _ ->
                val chosen = data.items
                    .filterIndexed { position, _ -> checked[position] }
                    .map { it.index }
                replyTo("display", SelectionData(chosen))
            }
            .show()
    }

    @Serializable
    data class MessageData(
        @SerialName("title") val title: String,
        @SerialName("searchPlaceholder") val searchPlaceholder: String,
        @SerialName("items") val items: List<Item>
    )

    @Serializable
    data class Item(
        @SerialName("title") val title: String,
        @SerialName("index") val index: Int,
        @SerialName("selected") val selected: Boolean
    )

    @Serializable
    data class SelectionData(
        @SerialName("selectedIndexes") val selectedIndexes: List<Int>
    )
}
```

> **This is the guide's first deliberate platform asymmetry, and it is worth naming.** iOS gets a searchable list screen; Android gets a scrolling multi-choice dialog with no search, because `setMultiChoiceItems` is ~15 lines and a searchable Android equivalent is a `DialogFragment` plus a `RecyclerView` plus an adapter plus two layout XML files — roughly 250 lines for a feature most option lists don't need.
>
> Ship this version first. Add the Android search screen when your longest list actually crosses the threshold where scrolling hurts, which in practice is around 30 options. 11.11 sketches it. The `searchPlaceholder` field is already in the contract precisely so that upgrade needs no contract change.

## 11.9 Verify

1. Browser: the `<select multiple>` is visible and usable; the summary button is not rendered.
2. App: the `<select multiple>` is gone; the summary button shows the current selection, or the empty label.
3. Open, check two, Done → the summary updates and the underlying `<select>` has both options selected.
4. Submit → the server receives both ids.
5. Open, **uncheck everything**, Done → the server receives the empty value, and the record's association is cleared. This is the case the hidden field exists for, and the one that breaks when someone "cleans up" the form markup.
6. Open, Cancel → nothing changes.

## 11.10 Fallback

`this.enabled` is false → `.bridge-native-only` stays `display: none`, the select is never hidden, and `open()` returns immediately even if something calls it. The page is a plain Rails form with a multiple select.

## 11.11 Variations

- **Android search screen** — replace the dialog with a `DialogFragment` showing `SearchView` + `RecyclerView`, holding the same `checked` array. The bridge contract is unchanged; `searchPlaceholder` is already being sent.
- **Grouped options** — add `group: String?` to `Item`. iOS becomes a sectioned table (natural); Android needs the RecyclerView variant, since dialogs can't do sections.
- **Min/max selection** — add `minSelected` / `maxSelected` to the payload and disable Done outside the range. Enforce it in the model as well; Chapter 1's rule applies.
- **Remote options** — if the list is too large to send in one message, you want Chapter 14, not this chapter.

## 11.12 Gotchas

1. **The summary button submits the form.** Missing `type="button"`.
2. **Clearing the selection doesn't persist.** The hidden empty-value field was removed, so the params omit the key and Rails leaves the association untouched.
3. **Crash on Done after a Turbo Stream update.** Missing the `isConnected` guard in `#applySelection`.
4. **Search hides selected items and they get lost on Done.** You filtered `selected` alongside `visible`. Keep selection in source-index terms; only `visible` is filtered.
5. **VoiceOver doesn't announce checked state.** `accessoryType` is visual; you also need `.selected` in `accessibilityTraits`.

---

# Chapter 12 — Date range

*Complexity 3 · ~85 lines JS, ~200 Swift, ~110 Kotlin*

Two date fields that constrain each other. The chapter exists mostly because of a platform asymmetry that you cannot design around.

## 12.1 The web problem

A check-in/check-out pair, a reporting period, a job window. On the web this is two `<input type="date">` elements plus:

- JS keeping `end.min` in sync with `start.value`
- a "clear end when start moves past it" rule
- client-side messaging for "end before start"
- and usually a dual-calendar range picker library, because two separate calendars feel wrong for one continuous concept

The last item is the expensive one: dual-calendar pickers are among the heaviest and most bug-prone widgets on the web.

## 12.2 The asymmetry

> **Android has a first-party date range picker. iOS does not.**
>
> `MaterialDatePicker.Builder.dateRangePicker()` returns a `Builder<Pair<Long, Long>>` and gives you a real range calendar for free. UIKit has `UIDatePicker`, which selects exactly one date, and no range equivalent.

So the two implementations are genuinely different shapes, and the contract has to be the thing they agree on rather than the UI. iOS gets a sheet with two compact pickers that constrain each other; Android gets the range calendar. Both produce the same two strings.

This is worth saying out loud because the instinct is to force symmetry — either by building a custom range calendar on iOS, or by degrading Android to two pickers. Both are wrong. The user on each platform should get their platform's answer, and the contract is what keeps that from leaking into Rails.

## 12.3 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ title: String, start: String?, end: String?, min: String?, max: String? }` |
| Native → Web | reply to `display` | `{ start: String, end: String }` |

All dates are `YYYY-MM-DD`, UTC, exactly as in Chapter 10 — and for exactly the same reasons. Both fields in the reply are always present; a range is not a range until both ends exist, so the native side does not enable Done until the user has chosen both.

## 12.4 Rails

```erb
<%# app/views/work_orders/_form.html.erb %>
<div data-controller="bridge--date-range"
     data-bridge--date-range-title-value="Job window">

  <div class="field-pair">
    <%= form.label :starts_on %>
    <%= form.date_field :starts_on,
          min: Date.current,
          max: 1.year.from_now.to_date,
          data: { "bridge--date-range-target": "start" } %>
  </div>

  <div class="field-pair">
    <%= form.label :ends_on %>
    <%= form.date_field :ends_on,
          min: Date.current,
          max: 1.year.from_now.to_date,
          data: { "bridge--date-range-target": "end" } %>
  </div>
</div>
```

```ruby
# app/models/work_order.rb
class WorkOrder < ApplicationRecord
  validates :starts_on, :ends_on, presence: true
  validate :ends_on_after_starts_on

  private

  def ends_on_after_starts_on
    return if starts_on.blank? || ends_on.blank?
    return if ends_on >= starts_on

    errors.add(:ends_on, "must be on or after the start date")
  end
end
```

The ordering rule lives in the model, not in the picker. The picker makes it hard to produce an invalid range; the model makes it impossible to save one. A native control that "guarantees" ordering is still only a UI, and a browser posting the same form has no such guarantee.

## 12.5 Web

```js
// app/javascript/controllers/bridge/date_range_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "date-range"
  static targets = ["start", "end"]
  static values = { title: { type: String, default: "Select dates" } }

  startTargetConnected(input) { this.#takeOver(input) }
  endTargetConnected(input) { this.#takeOver(input) }

  startTargetDisconnected(input) { this.#release(input) }
  endTargetDisconnected(input) { this.#release(input) }

  #takeOver(input) {
    if (!this.enabled) return

    input.readOnly = true
    input.setAttribute("aria-haspopup", "dialog")
    input.addEventListener("mousedown", this.#openNativePicker)
  }

  #release(input) {
    input.readOnly = false
    input.removeEventListener("mousedown", this.#openNativePicker)
  }

  // Either field opens the same range picker — the range is one concept.
  #openNativePicker = (event) => {
    event.preventDefault()

    this.send("display", {
      title: this.titleValue,
      start: this.startTarget.value || null,
      end: this.endTarget.value || null,
      min: this.startTarget.min || null,
      max: this.endTarget.max || null
    }, (reply) => {
      this.#applyRange(reply.data.start, reply.data.end)
    })
  }

  #applyRange(start, end) {
    if (!this.element.isConnected || !start || !end) return

    this.#assign(this.startTarget, start)
    this.#assign(this.endTarget, end)
  }

  #assign(input, value) {
    if (input.value === value) return

    input.value = value
    input.dispatchEvent(new Event("input", { bubbles: true }))
    input.dispatchEvent(new Event("change", { bubbles: true }))
  }
}
```

Tapping *either* field opens the range picker. That is the point of moving the pair into one component: on the web they are two controls that happen to be related, and in the app they are one control with two outputs.

## 12.6 iOS

```swift
// ios/MyApp/Bridge/DateRangeComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Presents a two-field sheet for a pair of web date inputs and replies
/// with both ends of the range as ISO-8601 calendar dates.
final class DateRangeComponent: BridgeComponent {
    override class var name: String { "date-range" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            presentSheet(with: data)
        }
    }

    // MARK: Private

    static let formatter: DateFormatter = {
        let formatter = DateFormatter()
        formatter.calendar = Calendar(identifier: .gregorian)
        formatter.locale = Locale(identifier: "en_US_POSIX")
        formatter.timeZone = TimeZone(identifier: "UTC")
        formatter.dateFormat = "yyyy-MM-dd"
        return formatter
    }()

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func presentSheet(with data: MessageData) {
        guard let viewController else { return }

        let sheet = DateRangeSheetViewController(
            sheetTitle: data.title,
            start: data.start.flatMap(Self.formatter.date(from:)),
            end: data.end.flatMap(Self.formatter.date(from:)),
            minimum: data.min.flatMap(Self.formatter.date(from:)),
            maximum: data.max.flatMap(Self.formatter.date(from:))
        ) { [unowned self] start, end in
            reply(
                to: Event.display.rawValue,
                with: RangeData(
                    start: Self.formatter.string(from: start),
                    end: Self.formatter.string(from: end)
                )
            )
        }

        let navigationController = UINavigationController(rootViewController: sheet)
        if let presentation = navigationController.sheetPresentationController {
            presentation.detents = [.medium()]
            presentation.prefersGrabberVisible = true
        }

        viewController.present(navigationController, animated: true)
    }
}

// MARK: Events

private extension DateRangeComponent {
    enum Event: String {
        case display
    }
}

// MARK: Message data

private extension DateRangeComponent {
    struct MessageData: Decodable {
        let title: String
        let start: String?
        let end: String?
        let min: String?
        let max: String?
    }

    struct RangeData: Encodable {
        let start: String
        let end: String
    }
}
```

```swift
// ios/MyApp/Bridge/DateRangeSheetViewController.swift
import UIKit

/// Two compact date pickers that constrain each other, with Cancel and Done.
final class DateRangeSheetViewController: UIViewController {
    private let startPicker = UIDatePicker()
    private let endPicker = UIDatePicker()
    private let onDone: (Date, Date) -> Void

    init(sheetTitle: String,
         start: Date?,
         end: Date?,
         minimum: Date?,
         maximum: Date?,
         onDone: @escaping (Date, Date) -> Void) {
        self.onDone = onDone
        super.init(nibName: nil, bundle: nil)

        title = sheetTitle

        for picker in [startPicker, endPicker] {
            picker.datePickerMode = .date
            picker.preferredDatePickerStyle = .compact
            picker.calendar = Calendar(identifier: .gregorian)
            picker.timeZone = TimeZone(identifier: "UTC")
            picker.minimumDate = minimum
            picker.maximumDate = maximum
        }

        let today = Date()
        startPicker.date = start ?? minimum ?? today
        endPicker.date = end ?? startPicker.date
        endPicker.minimumDate = startPicker.date
    }

    @available(*, unavailable)
    required init?(coder: NSCoder) {
        fatalError("init(coder:) is not supported")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        view.backgroundColor = .systemBackground

        navigationItem.leftBarButtonItem = UIBarButtonItem(
            systemItem: .cancel,
            primaryAction: UIAction { [unowned self] _ in dismiss(animated: true) }
        )

        navigationItem.rightBarButtonItem = UIBarButtonItem(
            systemItem: .done,
            primaryAction: UIAction { [unowned self] _ in
                onDone(startPicker.date, endPicker.date)
                dismiss(animated: true)
            }
        )

        startPicker.addAction(
            UIAction { [unowned self] _ in startDateChanged() },
            for: .valueChanged
        )

        let stack = UIStackView(arrangedSubviews: [
            row(title: String(localized: "Start"), picker: startPicker),
            row(title: String(localized: "End"), picker: endPicker)
        ])
        stack.axis = .vertical
        stack.spacing = 16
        stack.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(stack)

        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 24),
            stack.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 20),
            stack.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -20)
        ])
    }

    // MARK: Private

    private func startDateChanged() {
        endPicker.minimumDate = startPicker.date

        // Pull the end date forward rather than letting it go invalid.
        if endPicker.date < startPicker.date {
            endPicker.setDate(startPicker.date, animated: true)
        }
    }

    private func row(title: String, picker: UIDatePicker) -> UIView {
        let label = UILabel()
        label.text = title
        label.font = .preferredFont(forTextStyle: .body)
        label.adjustsFontForContentSizeCategory = true

        let row = UIStackView(arrangedSubviews: [label, UIView(), picker])
        row.axis = .horizontal
        row.alignment = .center
        row.spacing = 12

        picker.accessibilityLabel = title

        return row
    }
}
```

`startDateChanged` is the whole ordering rule, and it does the kind thing: it moves the end date rather than clearing it or showing an error. A user who slides the start date past the end almost always means "shift the window," not "discard my end date."

## 12.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/DateRangeComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import androidx.core.util.Pair
import com.google.android.material.datepicker.CalendarConstraints
import com.google.android.material.datepicker.CompositeDateValidator
import com.google.android.material.datepicker.DateValidatorPointBackward
import com.google.android.material.datepicker.DateValidatorPointForward
import com.google.android.material.datepicker.MaterialDatePicker
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable
import java.time.Instant
import java.time.LocalDate
import java.time.ZoneOffset

/**
 * Presents the Material date range picker for a pair of web date inputs
 * and replies with both ends as ISO-8601 calendar dates.
 */
class DateRangeComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private companion object {
        const val TAG = "DateRangeComponent"
        const val MILLIS_PER_DAY = 86_400_000L
    }

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            else -> Log.w(TAG, "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val fragment = delegate.destination.fragment

        val builder = MaterialDatePicker.Builder.dateRangePicker()
            .setTitleText(data.title)

        val start = toUtcMillis(data.start)
        val end = toUtcMillis(data.end)
        if (start != null && end != null) {
            builder.setSelection(Pair(start, end))
        }

        buildConstraints(data)?.let { builder.setCalendarConstraints(it) }

        val picker = builder.build()

        picker.addOnPositiveButtonClickListener { selection ->
            val from = selection.first
            val to = selection.second
            if (from == null || to == null) return@addOnPositiveButtonClickListener

            replyTo("display", RangeData(toIsoDate(from), toIsoDate(to)))
        }

        picker.show(fragment.childFragmentManager, TAG)
    }

    private fun buildConstraints(data: MessageData): CalendarConstraints? {
        val validators = buildList {
            toUtcMillis(data.min)?.let { add(DateValidatorPointForward.from(it)) }
            toUtcMillis(data.max)?.let {
                add(DateValidatorPointBackward.before(it + MILLIS_PER_DAY))
            }
        }

        if (validators.isEmpty()) return null

        return CalendarConstraints.Builder()
            .setValidator(CompositeDateValidator.allOf(validators))
            .build()
    }

    private fun toUtcMillis(value: String?): Long? {
        if (value.isNullOrBlank()) return null
        return runCatching {
            LocalDate.parse(value).atStartOfDay(ZoneOffset.UTC).toInstant().toEpochMilli()
        }.onFailure { Log.w(TAG, "Unparseable date: $value") }.getOrNull()
    }

    private fun toIsoDate(millis: Long): String =
        Instant.ofEpochMilli(millis).atZone(ZoneOffset.UTC).toLocalDate().toString()

    @Serializable
    data class MessageData(
        @SerialName("title") val title: String,
        @SerialName("start") val start: String? = null,
        @SerialName("end") val end: String? = null,
        @SerialName("min") val min: String? = null,
        @SerialName("max") val max: String? = null
    )

    @Serializable
    data class RangeData(
        @SerialName("start") val start: String,
        @SerialName("end") val end: String
    )
}
```

Note `androidx.core.util.Pair`, not `kotlin.Pair`. The Material API takes the AndroidX one, and its components are **nullable** — hence the guard before replying. Importing the wrong `Pair` produces a type error that reads like a generics problem and wastes twenty minutes.

Everything else — the UTC conversions, the `+ MILLIS_PER_DAY` for an inclusive maximum, the `DialogFragment` rotation caveat from 10.7 — carries over from Chapter 10 unchanged.

## 12.8 Verify

1. Browser: two ordinary date fields, both usable, both constrained by their `min`/`max` attributes.
2. App: tapping **either** field opens the range picker, pre-populated with the current values.
3. Choose a range → both fields update together.
4. Submit → both dates persist, and they match what was chosen.
5. **iOS only:** set the start date past the end date → the end date moves with it rather than going invalid.
6. **Both:** submit with the end before the start by editing in a browser → the model validation rejects it. The picker is not the guard.
7. Device timezone at UTC+13 and UTC−10 → same two strings. Same test as 10.8, and just as mandatory.

## 12.9 Fallback

`this.enabled` is false → neither input is made read-only, no listeners bind, and both behave as ordinary date fields. The `min`/`max` attributes still apply. The cross-field ordering rule falls back to the model validation, which is where it always lived.

## 12.10 Variations

- **Nights / duration display** — send `nightsLabel` and show a computed count in the sheet's prompt (iOS) or title (Android). No contract change.
- **Presets** ("this week", "last 30 days") — bar buttons on the iOS sheet; Android's picker has no slot for them, so they belong on the web form for both platforms instead.
- **Open-ended ranges** — if `end` may legitimately be blank, the reply type becomes `{ start: String, end: String? }` and Done must be enabled with only a start. Decide this before writing native code; it is awkward to retrofit.

## 12.11 Gotchas

1. **Kotlin type error on `setSelection`.** Wrong `Pair` — import `androidx.core.util.Pair`.
2. **The maximum date can't be selected.** `DateValidatorPointBackward.before()` is exclusive; add a day.
3. **Only one field updates.** `#applyRange` returned early because one of `start`/`end` was missing from the reply. The contract requires both.
4. **The iOS end picker lets you go before the start.** `endPicker.minimumDate` wasn't updated in the `valueChanged` action.
5. **Range picker shows the wrong month on Android after rotation.** The `DialogFragment` was recreated and lost its listener — 10.7's caveat applies here too.

---

# Chapter 13 — Dependent selects

*Complexity 3 · ~30 lines JS, 0 Swift, 0 Kotlin*

The one chapter in the guide whose answer is **"you already have the tool, and it isn't a bridge component."**

## 13.1 The web problem

Country → region → city. Category → subcategory. Customer → site → asset. The traditional implementation:

```js
// The kind of thing this replaces — and it is worse than it looks
countrySelect.addEventListener("change", async (event) => {
  regionSelect.disabled = true
  regionSelect.innerHTML = "<option>Loading…</option>"

  const response = await fetch(`/regions?country_id=${event.target.value}`)
  const regions = await response.json()

  regionSelect.innerHTML = regions
    .map((r) => `<option value="${r.id}">${r.name}</option>`)
    .join("")
  regionSelect.disabled = false

  // …and now: what if two changes are in flight? what if the previously
  // selected region is still valid? what about the third level? what about
  // the server's own validation messages? what about XSS in r.name?
})
```

Every one of those trailing questions is a real bug that ships. The interpolation on the last line is a real XSS vector if any name is user-supplied.

## 13.2 Why this isn't a bridge component

Run it through the decision path from 21.2:

- **Is it possible in plain HTML?** No.
- **Is the difference purely markup that the native app needs differently?** No — both browsers and apps need the same behavior.
- **Does the native platform have a first-party control for this?** **No.** There is no "dependent picker" in UIKit or Material. You would be building the cascade yourself, in Swift and again in Kotlin, with the same race conditions — plus a third implementation on the web for browsers.

The decision path stops there: *keep it on the web*. A hand-built native cascade has all the cost of a bridge component and none of the payoff.

What makes the cascade worth covering is that Rails already solves it with **zero JavaScript beyond three lines**, and the solution composes cleanly with the Chapter 9 select component — so the app still gets native pickers at every level of the cascade. You get the native feel without a native cascade.

## 13.3 The Turbo Frame solution

The server owns the cascade. It already knows which regions belong to a country; let it render them.

```ruby
# config/routes.rb
resources :work_orders do
  collection do
    get :location_fields
  end
end
```

```ruby
# app/controllers/work_orders_controller.rb
class WorkOrdersController < ApplicationController
  def location_fields
    @work_order = WorkOrder.new(
      country_id: params[:country_id],
      region_id: params[:region_id]
    )

    render partial: "work_orders/location_fields_frame",
           locals: { work_order: @work_order }
  end
end
```

```erb
<%# app/views/work_orders/_location_fields_frame.html.erb %>
<%= fields model: work_order do |form| %>
  <%= render "work_orders/location_fields", form: form, work_order: work_order %>
<% end %>
```

> **`fields`, not `form_with`.** The frame lives *inside* the page's existing `<form>`, so a frame response that rendered `form_with` would inject a second `<form>` inside the first. Nested forms are invalid HTML, and browsers resolve them by silently discarding the inner one — the fields would render, look correct, and never submit. Rails' `fields` helper yields the same form builder with no `<form>` tag, which is exactly what a fragment rendered into an existing form needs.

```erb
<%# app/views/work_orders/_location_fields.html.erb %>
<%= turbo_frame_tag "location_fields",
      data: {
        controller: "cascade",
        cascade_url_value: location_fields_work_orders_path
      } do %>

  <div data-controller="bridge--select"
       data-bridge--select-title-value="Country">
    <%= form.label :country_id, "Country" %>
    <%= form.collection_select :country_id, Country.order(:name), :id, :name,
          { include_blank: "Select a country" },
          {
            data: {
              "bridge--select-target": "select",
              action: "change->cascade#reload"
            }
          } %>
  </div>

  <% regions = work_order.country&.regions&.order(:name) || Region.none %>

  <div data-controller="bridge--select"
       data-bridge--select-title-value="Region">
    <%= form.label :region_id, "Region" %>
    <%= form.collection_select :region_id, regions, :id, :name,
          { include_blank: regions.any? ? "Select a region" : "Choose a country first" },
          {
            data: { "bridge--select-target": "select" },
            disabled: regions.empty?
          } %>
  </div>
<% end %>
```

```js
// app/javascript/controllers/cascade_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { url: String }

  reload(event) {
    // Send only the fields inside this frame, not the whole form.
    const params = new URLSearchParams()

    this.element.querySelectorAll("select[name], input[name]").forEach((field) => {
      const key = field.name.replace(/^\w+\[(\w+)\]$/, "$1")
      if (field.value) params.set(key, field.value)
    })

    this.element.src = `${this.urlValue}?${params}`
  }
}
```

> **Scope the parameters to the frame.** Serializing the whole form — `new FormData(form)` — looks convenient and is wrong for three reasons: a file input makes `URLSearchParams` produce `"[object File]"`, a long form can exceed URL length limits, and every unrelated keystroke ends up in your server logs and your CDN cache keys. The cascade needs the fields it depends on and nothing else.

That is the whole implementation. Ten lines of JavaScript, no fetch, no JSON, no innerHTML, no escaping concerns, no race handling — Turbo cancels an in-flight frame request when a new one starts, which is the race condition the hand-written version never gets right.

The properties you get for free are the interesting part:

| Concern | Hand-written JS | Turbo Frame |
|---|---|---|
| Escaping option labels | your problem | ERB does it |
| Two changes in flight | your problem | Turbo cancels the first |
| Server-side validation messages | separate code path | rendered with the fields |
| Third and fourth levels | multiplies | same partial, one more block |
| Loading state | manual | `[aria-busy]` on the frame |
| Works without JS | no | degrades to a full page request |

## 13.4 How it composes with the native picker

This is the part that matters for this guide: **each `<select>` in the cascade is still a Chapter 9 bridge component.** The native app gets a native picker for country *and* for region, and the cascade between them is server-rendered.

The mechanism that makes this work is Stimulus target lifecycle. Chapter 9's controller binds in `selectTargetConnected`, not in `connect`:

```js
selectTargetConnected(select) {
  if (!this.enabled) return
  select.addEventListener("mousedown", this.#openNativePicker)
  select.setAttribute("aria-haspopup", "listbox")
}
```

When Turbo replaces the frame's contents, the old `<select>` is removed and a new one is inserted. Stimulus fires `selectTargetDisconnected` for the old and `selectTargetConnected` for the new, and the native binding follows automatically. Had the binding been in `connect()`, the replaced select would have no listener and the native picker would silently stop opening for the second field only — a bug that is very hard to find by reading code.

> **This is why every component in this guide binds in `xTargetConnected` rather than `connect`.** It costs nothing when the DOM is static and it is the difference between working and not when a Turbo Frame or Stream is involved. Chapters 9, 10, 11 and 12 all follow this rule for this reason.

## 13.5 Verify

1. Browser, JS enabled: choosing a country replaces the region field with that country's regions.
2. Browser, **JS disabled**: the form still submits; the server validates the country/region pairing. Degraded, not broken.
3. App: both selects open native pickers.
4. App: choose a country in the native picker → the frame reloads → open the region picker → **it shows the new country's regions.** This is the test that proves the rebinding works.
5. App: change the country twice quickly → the region list matches the *second* country, not the first.
6. Submit a mismatched pair by crafting params directly → the model rejects it:

```ruby
# app/models/work_order.rb
validate :region_belongs_to_country

private

def region_belongs_to_country
  return if region.blank? || country.blank?
  return if region.country_id == country_id

  errors.add(:region_id, "isn't in the selected country")
end
```

Step 6 is not optional. The cascade is a convenience; the pairing rule is a model invariant. Anyone can post arbitrary params.

## 13.6 Gotchas

1. **The native picker stops working on the second select after a cascade.** The binding is in `connect()` instead of `selectTargetConnected`. This is the single most common way to break a bridge component inside a Turbo Frame.
2. **The dependent field isn't submitted.** `disabled: true` on a select means the browser omits it from the params entirely. That is usually what you want for "no country chosen yet", but if you need the key present, use a blank option and keep the field enabled.
3. **The frame reloads on every keystroke in an unrelated field.** `change->cascade#reload` is on the country select, but `change` bubbles — if you put the action on the frame instead of the select, every field in it triggers a reload. Scope the action to the specific select.
4. **The native picker is open when the frame swaps.** The reply arrives for a detached `<select>`. Chapter 11's `isConnected` guard applies here too; add it to the Chapter 9 controller if your selects live inside frames.
5. **Losing the rest of the form.** The frame only re-renders what is inside it. Keep the cascade's fields inside the frame and everything else outside, or reconstruct the full form builder in the frame response as shown in 13.3.

---

# Chapter 14 — Remote typeahead

*Complexity 4 · ~110 lines JS, ~230 Swift, ~200 Kotlin, ~25 lines Rails*

The highest-value component in the guide for most teams, and the one with the most interesting contract: the first where **the reply is a command, not an answer**, and the first with traffic flowing in both directions repeatedly.

## 14.1 Scope

This chapter assumes you already have a working web typeahead — Tom Select, a Stimulus autocomplete, whatever — backed by a JSON endpoint. It does not teach you to build one; that is ordinary web work and it is well covered elsewhere.

What it does is take that endpoint and reuse it, unchanged, to drive a native search screen. The web control and the native control end up sharing one query path, one authorization scope, and one set of results.

## 14.2 The web problem

A field backed by thousands of records — a customer, an asset, a part number — can't be a `<select>`. So it becomes a text input with search-as-you-type, and that means owning:

- debounce, and the off-by-one where the last keystroke's results never arrive
- request cancellation, and the race where an older response overwrites a newer one
- a result cache, and its invalidation
- match highlighting, empty states, loading states, error states
- keyboard navigation over a floating result list
- and on mobile, a result list that fights the virtual keyboard for space

## 14.3 The decision that shapes everything: who fetches?

There are two ways to build this, and the choice is not about elegance.

**Option A — native calls the JSON endpoint directly.** The Swift/Kotlin code does its own HTTP to `/technicians.json?q=…`.

**Option B — native asks the web layer, and the web layer fetches.** The native screen sends each query back over the bridge; the Stimulus controller runs the same `fetch` the web typeahead uses and sends the results back.

> **Choose option B.** The reason is authentication, not taste.
>
> The WebView owns the session. On iOS, `WKWebView` stores cookies in `WKHTTPCookieStore`, which `URLSession` does not share. On Android, the WebView's `CookieManager` is separate from OkHttp's cookie jar. So option A means either syncing cookies between two HTTP stacks on two platforms and keeping them in sync across session refresh, logout, and token rotation — or standing up a second, token-based authentication path for your API and maintaining its authorization rules alongside the ones your controllers already enforce.
>
> Option B costs one extra bridge round-trip per keystroke, which is an in-process message, not a network call. It is the cheaper side of a very lopsided trade.

Option B has a second benefit that shows up later: because the web layer does the fetching, scoping rules written in the controller — "technicians in the current account, active, not already assigned" — apply identically on both surfaces with no duplication.

## 14.4 The contract

This is the guide's first component where **one event carries several kinds of reply**, so the payload is a tagged union:

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `display` | `{ title, placeholder, minLength: Int, selectedLabel: String? }` |
| Native → Web | reply to `display` | `{ action: "search", query: String }` |
| Native → Web | reply to `display` | `{ action: "select", id: String, label: String }` |
| Native → Web | reply to `display` | `{ action: "clear" }` |
| Web → Native | `results` | `{ query: String, items: [{ id: String, label: String, sublabel: String? }] }` |

The `action` discriminator is what makes reply-many usable. Without it you would need three separate events and three separate `send` calls, and the callbacks would be registered against events that native has no reason to know about.

```
Web                                            Native
 │                                               │
 │ send("display", {title, placeholder, …})      │
 ├──────────────────────────────────────────────▶│ present search screen
 │                                               │
 │                              ┌────────────────┤ user types "sm"
 │ reply {action:"search",      │                │
 │        query:"sm"}           │                │
 │◀─────────────────────────────┘                │
 │ fetch /technicians.json?q=sm                  │
 │                                               │
 │ send("results", {query:"sm", items:[…]})      │
 ├──────────────────────────────────────────────▶│ if query is still current,
 │                                               │   reload the list
 │                              ┌────────────────┤ user types "smi"
 │ reply {action:"search",      │                │
 │        query:"smi"}          │                │
 │◀─────────────────────────────┘                │
 │        … repeat …                             │
 │                                               │
 │                              ┌────────────────┤ user taps a row
 │ reply {action:"select",      │                │
 │        id:"42", label:"…"}   │                │
 │◀─────────────────────────────┘                │ dismiss
 │ write hidden field, dispatch change           │
```

**Staleness is handled on the native side, by query string.** Native remembers the query it last asked for and ignores any `results` message whose `query` doesn't match. That is the whole race-condition fix, and it is three lines. The web side does not need to cancel anything for correctness — though it should still abort in-flight requests to save bandwidth.

## 14.5 Rails

The endpoint is the one your web typeahead already uses:

```ruby
# app/controllers/technicians_controller.rb
class TechniciansController < ApplicationController
  def index
    @technicians = Current.account
                          .technicians
                          .active
                          .search(params[:q].to_s)
                          .order(:name)
                          .limit(25)

    respond_to do |format|
      format.html
      format.json do
        render json: @technicians.map { |technician|
          {
            id: technician.id.to_s,
            label: technician.name,
            sublabel: technician.region_name
          }
        }
      end
    end
  end
end
```

Note `id` as a **string**. JSON numbers and form values are not the same type, and a hidden field's value is always a string; keeping the wire format a string everywhere removes a class of comparison bugs across three languages.

The form field:

```erb
<%# app/views/work_orders/_form.html.erb %>
<div data-controller="bridge--typeahead"
     data-bridge--typeahead-url-value="<%= technicians_path(format: :json) %>"
     data-bridge--typeahead-title-value="Find a technician"
     data-bridge--typeahead-placeholder-value="Name or region"
     data-bridge--typeahead-min-length-value="2"
     data-bridge--typeahead-empty-label-value="No technician assigned"
     data-bridge--typeahead-selected-label-value="<%= work_order.technician&.name %>">

  <%= form.label :technician_id, "Technician" %>

  <%= form.hidden_field :technician_id,
        data: { "bridge--typeahead-target": "value" } %>

  <%# Your existing web typeahead attaches to this input. %>
  <input type="text"
         class="typeahead-input"
         placeholder="Search technicians"
         value="<%= work_order.technician&.name %>"
         data-bridge--typeahead-target="input"
         data-bridge-hide-when-native>

  <button type="button"
          class="bridge-native-only field-summary"
          data-bridge--typeahead-target="summary"
          data-action="click->bridge--typeahead#open">
  </button>
</div>
```

The hidden field is the authoritative value on both surfaces. The text input is a browser affordance; the summary button is a native one. Only the hidden field is submitted.

> **This is the guide's strongest case for variant delivery (3.5).** As written, the native app receives the text input *and* whatever web typeahead library is bound to it — a 40–70 KB widget it initializes and then never shows. Splitting this into `_technician_field.html.erb` and `_technician_field.html+hotwire_native.erb` means the app gets the hidden field and the summary button, and your typeahead library never loads there at all.
>
> That is a bigger win than the markup saving. Everything else in this chapter — the contract, the JavaScript, the Swift, the Kotlin — is unchanged by the split.

### CSS

```css
/* app/assets/stylesheets/bridge.css */
[data-bridge-components~="typeahead"] [data-bridge-hide-when-native] {
  display: none;
}

[data-bridge-components~="typeahead"] .bridge-native-only {
  display: block;
}
```

Same shape as 11.6: hide the browser's control, reveal the native trigger. If you take the variant-partial route described above, both rules go away — the variant simply doesn't render the text input, and the plain partial doesn't render the summary button.

## 14.6 Web

```js
// app/javascript/controllers/bridge/typeahead_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "typeahead"
  static targets = ["value", "input", "summary"]
  static values = {
    url: String,
    title: { type: String, default: "Search" },
    placeholder: { type: String, default: "Search" },
    minLength: { type: Number, default: 2 },
    emptyLabel: { type: String, default: "Nothing selected" },
    selectedLabel: { type: String, default: "" }
  }

  connect() {
    super.connect()
    if (!this.enabled) return

    this.#renderSummary()
  }

  disconnect() {
    super.disconnect()
    this.#abort()
  }

  open() {
    if (!this.enabled) return

    this.send("display", {
      title: this.titleValue,
      placeholder: this.placeholderValue,
      minLength: this.minLengthValue,
      selectedLabel: this.#currentLabel() || null
    }, (reply) => {
      this.#handle(reply.data)
    })
  }

  // The native screen sends commands, not answers.
  #handle(data) {
    if (!data || !this.element.isConnected) return

    switch (data.action) {
      case "search": this.#search(data.query); break
      case "select": this.#select(data.id, data.label); break
      case "clear":  this.#select("", ""); break
    }
  }

  async #search(query) {
    this.#abort()
    this.abortController = new AbortController()

    const url = `${this.urlValue}?q=${encodeURIComponent(query)}`

    try {
      const response = await fetch(url, {
        headers: { Accept: "application/json" },
        signal: this.abortController.signal
      })

      if (!response.ok) throw new Error(`${response.status}`)

      const items = await response.json()
      this.send("results", { query: query, items: items })
    } catch (error) {
      if (error.name === "AbortError") return

      // Tell the native screen the search finished with nothing,
      // so it shows an empty state instead of a spinner forever.
      this.send("results", { query: query, items: [] })
    }
  }

  #select(id, label) {
    this.valueTarget.value = id
    this.selectedLabelValue = label
    if (this.hasInputTarget) this.inputTarget.value = label

    this.valueTarget.dispatchEvent(new Event("input", { bubbles: true }))
    this.valueTarget.dispatchEvent(new Event("change", { bubbles: true }))

    this.#renderSummary()
  }

  // The text input is the label source in the browser — but it does not exist
  // when this field is delivered as a variant partial (3.5, and the callout in
  // 14.5). `selectedLabelValue` is what the app has to fall back to, so Rails
  // must always render it.
  #currentLabel() {
    if (this.hasInputTarget && this.inputTarget.value) return this.inputTarget.value
    return this.selectedLabelValue
  }

  #renderSummary() {
    if (!this.hasSummaryTarget) return

    const label = this.#currentLabel()
    this.summaryTarget.textContent = label || this.emptyLabelValue
    this.summaryTarget.classList.toggle("is-empty", !label)
  }

  #abort() {
    if (this.abortController) {
      this.abortController.abort()
      this.abortController = null
    }
  }
}
```

The `catch` block is doing something specific and easy to omit: **a failed search still sends a `results` message.** If it didn't, the native screen would sit on a loading state forever with no way to recover. Every path that native is waiting on must terminate — the same principle as the confirm dialog in Chapter 7 replying on cancel.

## 14.7 iOS

```swift
// ios/MyApp/Bridge/TypeaheadComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Presents a native search screen backed by the web app's own JSON endpoint.
/// Queries are round-tripped through the bridge so the WebView's session is used.
final class TypeaheadComponent: BridgeComponent {
    override class var name: String { "typeahead" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .display:
            guard let data: MessageData = message.data() else { return }
            presentSearch(with: data)
        case .results:
            guard let data: ResultsData = message.data() else { return }
            searchViewController?.apply(results: data.items, for: data.query)
        }
    }

    // MARK: Private

    private weak var searchViewController: TypeaheadSearchViewController?

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func presentSearch(with data: MessageData) {
        guard let viewController else { return }

        let search = TypeaheadSearchViewController(
            screenTitle: data.title,
            placeholder: data.placeholder,
            minLength: data.minLength,
            selectedLabel: data.selectedLabel,
            onQueryChanged: { [unowned self] query in
                reply(to: Event.display.rawValue, with: Command.search(query: query))
            },
            onSelect: { [unowned self] item in
                reply(
                    to: Event.display.rawValue,
                    with: Command.select(id: item.id, label: item.label)
                )
            },
            onClear: { [unowned self] in
                reply(to: Event.display.rawValue, with: Command.clear())
            }
        )

        searchViewController = search

        let navigationController = UINavigationController(rootViewController: search)
        navigationController.modalPresentationStyle = .formSheet
        viewController.present(navigationController, animated: true)
    }
}

// MARK: Events

private extension TypeaheadComponent {
    enum Event: String {
        case display
        case results
    }
}

// MARK: Message data

extension TypeaheadComponent {
    struct Item: Decodable {
        let id: String
        let label: String
        let sublabel: String?
    }
}

private extension TypeaheadComponent {
    struct MessageData: Decodable {
        let title: String
        let placeholder: String
        let minLength: Int
        let selectedLabel: String?
    }

    struct ResultsData: Decodable {
        let query: String
        let items: [Item]
    }

    /// The tagged reply payload from 14.4.
    struct Command: Encodable {
        let action: String
        let query: String?
        let id: String?
        let label: String?

        static func search(query: String) -> Command {
            Command(action: "search", query: query, id: nil, label: nil)
        }

        static func select(id: String, label: String) -> Command {
            Command(action: "select", query: nil, id: id, label: label)
        }

        static func clear() -> Command {
            Command(action: "clear", query: nil, id: nil, label: nil)
        }
    }
}
```

```swift
// ios/MyApp/Bridge/TypeaheadSearchViewController.swift
import UIKit

/// A search screen that asks the web layer for results and renders whatever comes back.
final class TypeaheadSearchViewController: UITableViewController, UISearchResultsUpdating {
    private let placeholder: String
    private let minLength: Int
    private let selectedLabel: String?
    private let onQueryChanged: (String) -> Void
    private let onSelect: (TypeaheadComponent.Item) -> Void
    private let onClear: () -> Void

    private var items: [TypeaheadComponent.Item] = []
    private var pendingQuery: String?
    private var currentQuery = ""

    init(screenTitle: String,
         placeholder: String,
         minLength: Int,
         selectedLabel: String?,
         onQueryChanged: @escaping (String) -> Void,
         onSelect: @escaping (TypeaheadComponent.Item) -> Void,
         onClear: @escaping () -> Void) {
        self.placeholder = placeholder
        self.minLength = minLength
        self.selectedLabel = selectedLabel
        self.onQueryChanged = onQueryChanged
        self.onSelect = onSelect
        self.onClear = onClear

        super.init(style: .insetGrouped)
        title = screenTitle
    }

    @available(*, unavailable)
    required init?(coder: NSCoder) {
        fatalError("init(coder:) is not supported")
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        tableView.register(UITableViewCell.self, forCellReuseIdentifier: "item")
        tableView.keyboardDismissMode = .onDrag

        let search = UISearchController(searchResultsController: nil)
        search.searchResultsUpdater = self
        search.obscuresBackgroundDuringPresentation = false
        search.searchBar.placeholder = placeholder
        navigationItem.searchController = search
        navigationItem.hidesSearchBarWhenScrolling = false

        navigationItem.leftBarButtonItem = UIBarButtonItem(
            systemItem: .cancel,
            primaryAction: UIAction { [unowned self] _ in dismiss(animated: true) }
        )

        if selectedLabel != nil {
            navigationItem.rightBarButtonItem = UIBarButtonItem(
                title: String(localized: "Clear"),
                primaryAction: UIAction { [unowned self] _ in
                    onClear()
                    dismiss(animated: true)
                }
            )
        }

        updatePrompt()
        DispatchQueue.main.async { search.searchBar.becomeFirstResponder() }
    }

    /// Called by the component when a `results` message arrives.
    func apply(results: [TypeaheadComponent.Item], for query: String) {
        // Ignore anything that isn't an answer to the question we last asked.
        guard query == currentQuery else { return }

        items = results
        pendingQuery = nil
        tableView.reloadData()
        updatePrompt()
    }

    // MARK: Search

    func updateSearchResults(for searchController: UISearchController) {
        let query = searchController.searchBar.text?
            .trimmingCharacters(in: .whitespacesAndNewlines) ?? ""

        currentQuery = query

        guard query.count >= minLength else {
            items = []
            pendingQuery = nil
            tableView.reloadData()
            updatePrompt()
            return
        }

        pendingQuery = query
        updatePrompt()
        onQueryChanged(query)
    }

    // MARK: Table

    override func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        items.count
    }

    override func tableView(_ tableView: UITableView,
                            cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "item", for: indexPath)
        let item = items[indexPath.row]

        var configuration = cell.defaultContentConfiguration()
        configuration.text = item.label
        configuration.secondaryText = item.sublabel
        cell.contentConfiguration = configuration

        return cell
    }

    override func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        onSelect(items[indexPath.row])
        dismiss(animated: true)
    }

    // MARK: Private

    private func updatePrompt() {
        if pendingQuery != nil {
            navigationItem.prompt = String(localized: "Searching…")
        } else if currentQuery.count < minLength {
            navigationItem.prompt = String(
                localized: "Type at least \(minLength) characters"
            )
        } else if items.isEmpty {
            navigationItem.prompt = String(localized: "No matches")
        } else {
            navigationItem.prompt = nil
        }
    }
}
```

Note there is **no debounce on the native side.** `updateSearchResults` fires per keystroke and each fires a bridge message, which is an in-process call, not a network request. The debounce that matters belongs in the web layer, next to the `fetch` — and if you add one, `#search` in 14.6 is where it goes. Debouncing in both places produces a laggy field and is a common mistake when porting a web typeahead.

## 14.8 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/TypeaheadComponent.kt
package dev.hotwire.demo.bridge

import android.content.Context
import android.text.Editable
import android.text.TextWatcher
import android.util.Log
import android.view.ViewGroup
import android.widget.ArrayAdapter
import android.widget.EditText
import android.widget.LinearLayout
import android.widget.ListView
import android.widget.TextView
import androidx.appcompat.app.AlertDialog
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Presents a native search dialog backed by the web app's own JSON endpoint.
 * Queries are round-tripped through the bridge so the WebView's session is used.
 */
class TypeaheadComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private companion object {
        const val TAG = "TypeaheadComponent"
    }

    private var dialog: AlertDialog? = null
    private var listAdapter: ArrayAdapter<String>? = null
    private var statusView: TextView? = null
    private var items: List<Item> = emptyList()
    private var currentQuery = ""
    private var minLength = 2

    override fun onReceive(message: Message) {
        when (message.event) {
            "display" -> handleDisplayEvent(message)
            "results" -> handleResultsEvent(message)
            else -> Log.w(TAG, "Unknown event for message: $message")
        }
    }

    private fun handleDisplayEvent(message: Message) {
        val data = message.data<MessageData>() ?: return
        val context = delegate.destination.fragment.context ?: return

        minLength = data.minLength
        currentQuery = ""
        items = emptyList()

        val input = EditText(context).apply {
            hint = data.placeholder
            isSingleLine = true
        }

        val status = TextView(context).apply {
            setPadding(0, 16, 0, 16)
        }

        val adapter = ArrayAdapter<String>(
            context, android.R.layout.simple_list_item_1, mutableListOf()
        )

        val list = ListView(context).apply {
            this.adapter = adapter
            layoutParams = LinearLayout.LayoutParams(
                ViewGroup.LayoutParams.MATCH_PARENT, 0, 1f
            )
        }

        val container = LinearLayout(context).apply {
            orientation = LinearLayout.VERTICAL
            setPadding(48, 32, 48, 0)
            addView(input)
            addView(status)
            addView(list)
        }

        input.addTextChangedListener(object : TextWatcher {
            override fun afterTextChanged(s: Editable?) {
                onQueryChanged(s?.toString().orEmpty().trim(), context)
            }

            override fun beforeTextChanged(s: CharSequence?, a: Int, b: Int, c: Int) = Unit
            override fun onTextChanged(s: CharSequence?, a: Int, b: Int, c: Int) = Unit
        })

        list.setOnItemClickListener { _, _, position, _ ->
            items.getOrNull(position)?.let { item ->
                replyTo("display", Command.select(item.id, item.label))
                dismiss()
            }
        }

        val builder = MaterialAlertDialogBuilder(context)
            .setTitle(data.title)
            .setView(container)
            .setNegativeButton(android.R.string.cancel) { _, _ -> dismiss() }

        if (data.selectedLabel != null) {
            builder.setNeutralButton("Clear") { _, _ ->
                replyTo("display", Command.clear())
                dismiss()
            }
        }

        listAdapter = adapter
        statusView = status
        dialog = builder.show()

        updateStatus(context)
    }

    private fun handleResultsEvent(message: Message) {
        val data = message.data<ResultsData>() ?: return

        // Ignore anything that isn't an answer to the question we last asked.
        if (data.query != currentQuery) return

        items = data.items

        listAdapter?.apply {
            clear()
            addAll(data.items.map { listOf(it.label, it.sublabel).filterNotNull().joinToString(" — ") })
            notifyDataSetChanged()
        }

        delegate.destination.fragment.context?.let { updateStatus(it) }
    }

    private fun onQueryChanged(query: String, context: Context) {
        currentQuery = query

        if (query.length < minLength) {
            items = emptyList()
            listAdapter?.apply { clear(); notifyDataSetChanged() }
            updateStatus(context)
            return
        }

        updateStatus(context, searching = true)
        replyTo("display", Command.search(query))
    }

    private fun updateStatus(context: Context, searching: Boolean = false) {
        statusView?.text = when {
            searching -> "Searching…"
            currentQuery.length < minLength -> "Type at least $minLength characters"
            items.isEmpty() -> "No matches"
            else -> ""
        }
    }

    private fun dismiss() {
        dialog?.dismiss()
        dialog = null
        listAdapter = null
        statusView = null
    }

    @Serializable
    data class MessageData(
        @SerialName("title") val title: String,
        @SerialName("placeholder") val placeholder: String,
        @SerialName("minLength") val minLength: Int,
        @SerialName("selectedLabel") val selectedLabel: String? = null
    )

    @Serializable
    data class ResultsData(
        @SerialName("query") val query: String,
        @SerialName("items") val items: List<Item>
    )

    @Serializable
    data class Item(
        @SerialName("id") val id: String,
        @SerialName("label") val label: String,
        @SerialName("sublabel") val sublabel: String? = null
    )

    /** The tagged reply payload from 14.4. */
    @Serializable
    data class Command(
        @SerialName("action") val action: String,
        @SerialName("query") val query: String? = null,
        @SerialName("id") val id: String? = null,
        @SerialName("label") val label: String? = null
    ) {
        companion object {
            fun search(query: String) = Command("search", query = query)
            fun select(id: String, label: String) = Command("select", id = id, label = label)
            fun clear() = Command("clear")
        }
    }
}
```

> **This component holds state across messages**, which none of the earlier ones do. That state — `dialog`, `listAdapter`, `currentQuery` — is tied to a screen, and the component outlives individual messages but not necessarily the screen. `dismiss()` clears every reference, and it is called on all three exit paths. Leaking an `AlertDialog` reference past its fragment is a `WindowLeaked` crash in release builds.

## 14.9 Verify

1. Browser: the text input works with your existing web typeahead; the summary button isn't rendered.
2. App: the summary button shows the current selection; tapping it opens the search screen with the keyboard already up.
3. Type one character → "Type at least 2 characters", no request made.
4. Type two → results appear. Watch the Rails log: **one request per keystroke, carrying the session cookie**, hitting the same endpoint the browser uses.
5. Type quickly and then delete back to two characters → the list matches the *current* query, never an earlier one. This is the staleness guard; try to break it.
6. Select a row → the screen dismisses, the summary updates, and the hidden field holds the id.
7. Submit → the association persists.
8. **Turn off the network and search** → the screen shows "No matches" rather than spinning forever.
9. Cancel → nothing changes.

Step 4 is the one that proves the architecture: if you see a 401 or an empty result set there, something is fetching outside the WebView's session, and you have drifted into option A.

## 14.10 Fallback

`this.enabled` is false → the summary button stays hidden, the text input stays visible, and `open()` returns immediately. Your web typeahead is untouched, and so is the endpoint it calls.

## 14.11 Variations

- **Recent / suggested items** — send them in `display` as `initialItems` and render before any query is typed. No new events.
- **Create-if-missing** ("Add 'Smith' as a new technician") — add `{ action: "create", label }` to the tagged reply and have the web layer POST it. The union is already there; this is one more case.
- **Multi-select typeahead** — combine with Chapter 11: the reply becomes `{ action: "select", ids: [String] }` and the screen keeps checkmarks. Straightforward, but decide it up front — the single-select screen's dismiss-on-tap behavior is wrong for multi.
- **Paging** — add `{ action: "loadMore", query, offset }`. The web layer appends and re-sends `results` with the full list rather than a delta, which keeps the native side stateless about paging.

## 14.12 Gotchas

1. **Results from an old query overwrite new ones.** The `query == currentQuery` guard is missing, or it is comparing an untrimmed string on one side and a trimmed one on the other. Trim in exactly one place — the web layer — and echo the trimmed string back in `results`.
2. **The spinner never stops.** A `fetch` rejection path that doesn't send a `results` message. Every terminal state must be reported.
3. **401s in the Rails log.** Native is making the request itself. See 14.3.
4. **`WindowLeaked` on Android.** A dialog reference outlived its fragment. Clear it on every exit path.
5. **The field feels sluggish.** Debounce applied on both the native side and the web side. Keep it in one place, next to the network call.
6. **Selecting doesn't persist.** The `change` event was dispatched on the text input rather than the hidden field. Only the hidden field is submitted; dispatch on it.

---

# Chapter 15 — Repeating nested fieldsets

*Complexity 4 · ~90 lines JS, ~110 Swift, ~100 Kotlin, ~90 lines Rails*

Parts on a work order, line items on an invoice, attendees on an event. This is the most Rails-specific complexity in the guide — and the chapter where the honest answer is "most of this stays on the web."

## 15.1 The web problem

`accepts_nested_attributes_for` plus a dynamic add/remove UI has historically meant cocoon, or a hand-rolled equivalent:

- cloning a template and rewriting every `name` attribute's index
- generating unique indexes that don't collide with persisted records
- hiding a removed row and setting `_destroy` rather than deleting the DOM node
- keeping the "add" button's insertion point correct
- and re-running all of it when a validation error re-renders the form

## 15.2 What Rails gives you now

You do not need a dependency for this. `<template>` plus a ~40-line Stimulus controller covers it, and the indexing problem has a one-line answer:

```erb
<%# app/views/work_orders/_parts_fields.html.erb %>
<div data-controller="nested-form"
     data-nested-form-wrapper-selector-value=".nested-row">

  <div data-nested-form-target="rows">
    <%= form.fields_for :parts do |part_form| %>
      <%= render "part_fields", form: part_form %>
    <% end %>
  </div>

  <template data-nested-form-target="template">
    <%= form.fields_for :parts,
          WorkOrderPart.new,
          child_index: "NEW_RECORD" do |part_form| %>
      <%= render "part_fields", form: part_form %>
    <% end %>
  </template>

  <button type="button"
          class="button-secondary"
          data-nested-form-target="add"
          data-action="nested-form#add"
          data-bridge-hide-when-native>
    Add part
  </button>
</div>
```

```erb
<%# app/views/work_orders/_part_fields.html.erb %>
<div class="nested-row" data-nested-row>
  <%= form.hidden_field :id %>
  <%= form.hidden_field :_destroy, data: { nested_form_target: "destroy" } %>

  <div class="nested-row__fields">
    <%= form.label :name, "Part" %>
    <%= form.text_field :name %>

    <%= form.label :quantity %>
    <%= form.number_field :quantity, inputmode: "numeric", min: 1 %>
  </div>

  <button type="button"
          class="button-danger"
          data-action="nested-form#remove"
          data-nested-row-label="<%= form.object.name.presence || "this part" %>">
    Remove
  </button>
</div>
```

`child_index: "NEW_RECORD"` is the trick that removes the whole index-generation problem. Rails renders the template with a literal placeholder in every field name — `work_order[parts_attributes][NEW_RECORD][name]` — and the controller substitutes a timestamp when cloning. Timestamps never collide with database ids and never collide with each other within a single click.

```js
// app/javascript/controllers/nested_form_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["rows", "template", "destroy"]

  add(event) {
    event?.preventDefault()

    const html = this.templateTarget.innerHTML.replace(
      /NEW_RECORD/g,
      new Date().getTime().toString()
    )

    this.rowsTarget.insertAdjacentHTML("beforeend", html)
    this.#focusFirstField(this.rowsTarget.lastElementChild)
  }

  remove(event) {
    event.preventDefault()
    this.#removeRow(event.currentTarget.closest("[data-nested-row]"))
  }

  #removeRow(row) {
    if (!row) return

    const id = row.querySelector("input[name*='[id]']")

    if (id && id.value) {
      // Persisted: mark for destruction so Rails deletes it on save.
      row.querySelector("input[name*='_destroy']").value = "1"
      row.hidden = true
    } else {
      // Never saved: just drop it.
      row.remove()
    }
  }

  #focusFirstField(row) {
    row?.querySelector("input:not([type=hidden]), select, textarea")?.focus()
  }
}
```

```ruby
# app/models/work_order.rb
class WorkOrder < ApplicationRecord
  has_many :parts, class_name: "WorkOrderPart", dependent: :destroy

  accepts_nested_attributes_for :parts,
                                allow_destroy: true,
                                reject_if: :all_blank
end
```

```ruby
# app/controllers/work_orders_controller.rb
def work_order_params
  params.require(:work_order).permit(
    :title, :scheduled_on,
    parts_attributes: [:id, :name, :quantity, :_destroy]
  )
end
```

Note `row.hidden = true` rather than `display: none` via a class. The `hidden` property is what the platform and assistive technology both understand, and a hidden row's inputs are still submitted — which is exactly what `_destroy` needs.

## 15.3 What native should and shouldn't do here

Run the fields themselves through the decision path: is there a first-party native control for "a repeating set of text and number inputs"? No. Building one means rendering the rows natively, which means the row data lives in Swift and Kotlin, which breaks Chapter 1's rule outright. **The fields stay web.**

What is genuinely worth moving native is smaller and specific:

| Concern | Native worth it? | Why |
|---|---|---|
| The row's inputs | **No** | No native equivalent; would duplicate state |
| "Add part" button | **Yes** | It scrolls out of reach on a long list; the toolbar is always visible |
| Remove confirmation | **Yes** | Destructive actions deserve a native alert and error haptics |
| Reordering rows | Later | Real win, but it needs native row rendering — see Appendix B |

So this chapter's bridge component is deliberately small: it moves the add button into the chrome and puts a native destructive confirmation on removal. That is the whole native surface.

## 15.4 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `connect` | `{ addTitle: String }` |
| Native → Web | reply to `connect` | *(none)* — fires on every tap of the native Add button |
| Web → Native | `confirmRemove` | `{ title: String, message: String, confirmTitle: String, cancelTitle: String }` |
| Native → Web | reply to `confirmRemove` | `{ confirmed: Bool }` |

`connect` is reply-many, exactly like the submit button in Chapter 5. `confirmRemove` replies once per invocation, on both outcomes, for the reason given in Chapter 7.

## 15.5 Web

The bridge controller wraps the plain one rather than replacing it — the nested-form logic is identical on both surfaces:

```js
// app/javascript/controllers/bridge/nested_form_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "nested-form"
  static values = {
    addTitle: { type: String, default: "Add" },
    confirmTitle: { type: String, default: "Remove" },
    cancelTitle: { type: String, default: "Keep" }
  }

  connect() {
    super.connect()
    if (!this.enabled) return

    this.send("connect", { addTitle: this.addTitleValue }, () => {
      this.#plainController?.add()
    })
  }

  // Called instead of nested-form#remove when the bridge is available.
  remove(event) {
    event.preventDefault()

    const row = event.currentTarget.closest("[data-nested-row]")
    const label = event.currentTarget.dataset.nestedRowLabel || "this row"

    if (!this.enabled) {
      this.#plainController?.remove(event)
      return
    }

    this.send("confirmRemove", {
      title: this.confirmTitleValue,
      message: `Remove ${label}?`,
      confirmTitle: this.confirmTitleValue,
      cancelTitle: this.cancelTitleValue
    }, (reply) => {
      if (!reply.data.confirmed || !this.element.isConnected) return
      this.#plainController?.removeRowElement(row)
    })
  }

  get #plainController() {
    return this.application.getControllerForElementAndIdentifier(
      this.element,
      "nested-form"
    )
  }
}
```

That requires one small change to the plain controller — exposing the row removal so both callers share it:

```js
// app/javascript/controllers/nested_form_controller.js — replace #removeRow
removeRowElement(row) {
  if (!row) return

  const id = row.querySelector("input[name*='[id]']")

  if (id && id.value) {
    row.querySelector("input[name*='_destroy']").value = "1"
    row.hidden = true
  } else {
    row.remove()
  }
}

remove(event) {
  event.preventDefault()
  this.removeRowElement(event.currentTarget.closest("[data-nested-row]"))
}
```

And the markup carries both controllers, with the bridge one taking the action:

```erb
<%# app/views/work_orders/_parts_fields.html.erb — the wrapper element %>
<div data-controller="nested-form bridge--nested-form"
     data-bridge--nested-form-add-title-value="Add part"
     data-bridge--nested-form-confirm-title-value="Remove"
     data-bridge--nested-form-cancel-title-value="Keep">
```

```erb
<%# app/views/work_orders/_part_fields.html.erb — the remove button %>
<button type="button"
        class="button-danger"
        data-action="bridge--nested-form#remove"
        data-nested-row-label="<%= form.object.name.presence || "this part" %>">
  Remove
</button>
```

> **Two controllers on one element, one owning the behavior and one owning the platform, is a pattern worth reusing.** The plain controller has no knowledge of the bridge and is independently testable; the bridge controller adds native affordances and falls through to the plain one when `this.enabled` is false. Every component in this guide could be written this way; it earns its keep whenever the web behavior is substantial enough to deserve its own tests.

### CSS

```css
/* app/assets/stylesheets/bridge.css */
[data-bridge-components~="nested-form"] [data-bridge-hide-when-native] {
  display: none;
}
```

Only one rule here: the web "Add part" button is hidden because native puts the action in the chrome. The Remove buttons stay visible on both surfaces — native adds a confirmation to them rather than replacing them.

## 15.6 iOS

```swift
// ios/MyApp/Bridge/NestedFormComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Moves a nested form's "add row" action into the bottom toolbar and puts
/// a native destructive confirmation on row removal.
final class NestedFormComponent: BridgeComponent {
    override class var name: String { "nested-form" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .connect:
            guard let data: ConnectData = message.data() else { return }
            configureToolbar(addTitle: data.addTitle)
        case .confirmRemove:
            guard let data: ConfirmData = message.data() else { return }
            presentConfirmation(with: data)
        }
    }

    // MARK: Private

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func configureToolbar(addTitle: String) {
        guard let viewController else { return }

        let action = UIAction { [unowned self] _ in
            reply(to: Event.connect.rawValue)
        }

        let add = UIBarButtonItem(
            title: addTitle,
            image: UIImage(systemName: "plus"),
            primaryAction: action
        )

        viewController.toolbarItems = [.flexibleSpace(), add]
        viewController.navigationController?.setToolbarHidden(false, animated: true)
    }

    private func presentConfirmation(with data: ConfirmData) {
        guard let viewController else { return }

        let alert = UIAlertController(
            title: data.title,
            message: data.message,
            preferredStyle: .alert
        )

        alert.addAction(
            UIAlertAction(title: data.cancelTitle, style: .cancel) { [unowned self] _ in
                reply(to: Event.confirmRemove.rawValue, with: ConfirmReply(confirmed: false))
            }
        )

        alert.addAction(
            UIAlertAction(title: data.confirmTitle, style: .destructive) { [unowned self] _ in
                UINotificationFeedbackGenerator().notificationOccurred(.warning)
                reply(to: Event.confirmRemove.rawValue, with: ConfirmReply(confirmed: true))
            }
        )

        viewController.present(alert, animated: true)
    }
}

// MARK: Events

private extension NestedFormComponent {
    enum Event: String {
        case connect
        case confirmRemove
    }
}

// MARK: Message data

private extension NestedFormComponent {
    struct ConnectData: Decodable {
        let addTitle: String
    }

    struct ConfirmData: Decodable {
        let title: String
        let message: String
        let confirmTitle: String
        let cancelTitle: String
    }

    struct ConfirmReply: Encodable {
        let confirmed: Bool
    }
}
```

> **Chrome is a shared resource, and bridge components will fight over it.**
>
> Chapter 5's `FormComponent` owns `navigationItem.rightBarButtonItem`. If this component also claimed it, whichever `connect` arrived last would win, and the other button would silently vanish — on a page with both a submit button and a nested form, which is the normal case.
>
> That is why Add goes in the **bottom toolbar** here, not the navigation bar. Decide up front which component owns which slot — right bar button, left bar button, toolbar, overflow menu — and write it down. This is not a bug you find by reading one component's code.

## 15.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/NestedFormComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import android.view.Menu
import android.view.MenuItem
import androidx.appcompat.widget.Toolbar
import androidx.fragment.app.Fragment
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.demo.R
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Adds an "add row" action to the toolbar overflow and puts a native
 * destructive confirmation on row removal.
 */
class NestedFormComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private companion object {
        const val TAG = "NestedFormComponent"

        // Distinct from FormComponent's id so the two never collide.
        const val ADD_ITEM_ID = 41
    }

    private val fragment: Fragment
        get() = delegate.destination.fragment

    private val toolbar: Toolbar?
        get() = fragment.view?.findViewById(R.id.toolbar)

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnectEvent(message)
            "confirmRemove" -> handleConfirmRemoveEvent(message)
            else -> Log.w(TAG, "Unknown event for message: $message")
        }
    }

    private fun handleConnectEvent(message: Message) {
        val data = message.data<ConnectData>() ?: return
        val menu = toolbar?.menu ?: return

        // Always remove before adding — Turbo re-renders can fire connect twice.
        menu.removeItem(ADD_ITEM_ID)

        // Order 1 keeps it left of FormComponent's submit button (order 999).
        menu.add(Menu.NONE, ADD_ITEM_ID, 1, data.addTitle).apply {
            setShowAsAction(MenuItem.SHOW_AS_ACTION_NEVER)
            setOnMenuItemClickListener {
                replyTo("connect")
                true
            }
        }
    }

    private fun handleConfirmRemoveEvent(message: Message) {
        val data = message.data<ConfirmData>() ?: return
        val context = fragment.context ?: return

        MaterialAlertDialogBuilder(context)
            .setTitle(data.title)
            .setMessage(data.message)
            .setCancelable(true)
            .setNegativeButton(data.cancelTitle) { _, _ -> reply(false) }
            .setPositiveButton(data.confirmTitle) { _, _ -> reply(true) }
            .setOnCancelListener { reply(false) }
            .show()
    }

    private fun reply(confirmed: Boolean) {
        replyTo("confirmRemove", ConfirmReply(confirmed))
    }

    @Serializable
    data class ConnectData(
        @SerialName("addTitle") val addTitle: String
    )

    @Serializable
    data class ConfirmData(
        @SerialName("title") val title: String,
        @SerialName("message") val message: String,
        @SerialName("confirmTitle") val confirmTitle: String,
        @SerialName("cancelTitle") val cancelTitle: String
    )

    @Serializable
    data class ConfirmReply(
        @SerialName("confirmed") val confirmed: Boolean
    )
}
```

The `ADD_ITEM_ID = 41` / `FormComponent`'s `37` is the Android version of the same shared-chrome problem. Two components calling `menu.add` with the same id means one silently replaces the other. Keep the ids in one place — a shared `object MenuIds` is better than constants scattered across components.

## 15.8 Verify

1. Browser: "Add part" adds a row with empty fields and focuses the first one. Remove deletes an unsaved row outright and hides a saved one.
2. Browser: add three rows, fill them, submit → three parts persist with distinct attributes. This is the indexing test; a collision shows up as two rows merging into one.
3. Browser: remove a **saved** row, submit → it is destroyed. Check the SQL.
4. App: the web "Add part" button is gone; an Add action is in the bottom toolbar (iOS) or overflow menu (Android).
5. App: tap Add → a row appears in the web form and the first field takes focus.
6. App: tap Remove → a native destructive alert appears. Confirm → the row goes. Cancel → it stays.
7. App: **Android back-gesture the confirmation away** → the row stays and the page still responds.
8. App: on a page that also has a Chapter 5 submit button → **both** are visible and both work. This is the shared-chrome regression test.
9. Trigger a validation error so the form re-renders with errors → the rows come back with their values, and add/remove still work. Stimulus reconnects; the native toolbar is reconfigured by the fresh `connect`.

## 15.9 Fallback

`this.enabled` is false → the bridge controller's `connect` sends nothing, `remove` falls through to the plain controller's logic, and the web "Add part" button is never hidden. The nested form is a plain Rails nested form with 40 lines of Stimulus, which is what it is on every browser.

## 15.10 Variations & what's deferred

- **Row reordering** — the real remaining win, and the reason Appendix B still lists it. Doing it properly means native row rendering, which puts row order in native state; the honest design has the web form own an explicit `position` field that native writes. Deferred.
- **A native keyboard accessory** with Previous / Next / Done walking focus across every field in the form. On a twelve-row nested form this is arguably a bigger win than anything in this chapter. It is deferred because `WKWebView`'s own input accessory view cannot be replaced through public API — the workable approach is a `UIToolbar` tracking keyboard-frame notifications, with an IME-inset equivalent on Android, and it deserves a full chapter rather than a footnote.
- **Row limits** — send `maxRows` in `connect` and disable the native Add button at the limit. Validate `parts.size` in the model too.

## 15.11 Gotchas

1. **Two new rows share field names.** `new Date().getTime()` returned the same millisecond for two rapid clicks, or you used the array length as the index and it collided with a removed row. Timestamps are safer than counters; if you add rows programmatically in a loop, add a counter suffix.
2. **Removing a saved row does nothing.** The `_destroy` hidden field isn't permitted in strong params, or `allow_destroy: true` is missing from `accepts_nested_attributes_for`.
3. **Removed rows come back after a validation error.** You removed the DOM node for a persisted record instead of setting `_destroy`. The server never heard about the removal.
4. **The native Add button replaces the submit button.** Both components claimed the same navigation slot. See the callout in 15.6.
5. **The toolbar stays visible after leaving the screen.** `setToolbarHidden(false, …)` applies to the navigation controller, not the screen. Hide it again when the component's destination goes away, or accept it on all screens.
6. **`reject_if: :all_blank` silently discards a row.** It is doing its job — a row where every attribute is blank is dropped. If a row is legitimately all-blank-but-meaningful, use a lambda that checks the specific attribute instead.

---

# Chapter 16 — Multi-step forms

*Complexity 4 · ~40 lines JS, ~80 Swift, ~90 Kotlin, ~140 lines Rails*

The chapter with the largest Rails section and the smallest native one — because getting the Rails architecture right removes almost all of the native work.

## 16.1 The web problem

A four-step intake flow, built the usual way, is a client-side state machine:

- a `currentStep` variable and show/hide logic over four `<div>`s
- per-step validation duplicated from the model
- a progress bar driven by that variable
- browser-back interception, because back should go to step 2, not to the previous page
- draft persistence to `localStorage`, because a refresh loses everything
- and a final submit that posts all four steps at once, so any server-side error lands the user back at the top with no indication of which step it came from

That is several hundred lines, and every one of those bullets is a thing that breaks.

## 16.2 The design decision: steps are URLs

> **Each step is a real URL, a real controller action, and a real Turbo visit.**

This is the whole chapter. The consequences are large and they are mostly about what you *stop* having to build:

| Concern | Client-side wizard | Steps as URLs |
|---|---|---|
| Progress | your variable | `STEPS.index(params[:id])` |
| Back | intercept `popstate` | the browser back button, free |
| **Native back** | **doesn't work at all** | **the native back stack, free** |
| Refresh mid-flow | lose everything, or `localStorage` | the record is already saved |
| Per-step validation | duplicated in JS | validation contexts |
| Deep link to step 3 | impossible | it's a URL |
| Resuming next week | build it | it's a URL |

The native row is the one that matters for this guide. A Hotwire Native app's back button pops the navigation stack, and the navigation stack is made of visited URLs. A client-side wizard has one URL, so native back leaves the entire flow — from step 3 straight out of the form. There is no bridge component that fixes that; it is a consequence of the architecture. Steps-as-URLs makes native back correct with no native code at all.

## 16.3 Rails

```ruby
# config/routes.rb
resources :work_orders do
  resources :steps, only: [:show, :update], module: :work_orders
end
```

```ruby
# app/controllers/work_orders/steps_controller.rb
module WorkOrders
  class StepsController < ApplicationController
    STEPS = %w[details schedule parts review].freeze

    before_action :set_work_order
    before_action :set_step

    def show
      render @step
    end

    def update
      @work_order.assign_attributes(work_order_params)

      if @work_order.save(context: @step.to_sym)
        redirect_to next_location
      else
        render @step, status: :unprocessable_entity
      end
    end

    private

    def set_work_order
      @work_order = Current.account.work_orders.find(params[:work_order_id])
    end

    def set_step
      @step = params[:id]
      redirect_to work_order_step_path(@work_order, STEPS.first) unless STEPS.include?(@step)
    end

    def next_step
      STEPS[STEPS.index(@step) + 1]
    end

    def next_location
      return work_order_step_path(@work_order, next_step) if next_step

      flash[:notice] = "Work order submitted"
      work_order_path(@work_order)
    end

    def work_order_params
      params.require(:work_order).permit(
        :title, :notes, :scheduled_on, :starts_on, :ends_on,
        parts_attributes: [:id, :name, :quantity, :_destroy]
      )
    end

    helper_method :step_number, :step_count

    def step_number = STEPS.index(@step) + 1
    def step_count  = STEPS.size
  end
end
```

Per-step validation uses **validation contexts**, so the model stays the single source of truth and each step validates only what it collects:

```ruby
# app/models/work_order.rb
class WorkOrder < ApplicationRecord
  has_many :parts, class_name: "WorkOrderPart", dependent: :destroy
  accepts_nested_attributes_for :parts, allow_destroy: true, reject_if: :all_blank

  # Always true, whatever the step.
  validates :account, presence: true

  # Step-scoped.
  validates :title, presence: true, length: { maximum: 120 }, on: :details
  validates :scheduled_on, presence: true, on: :schedule
  validate  :ends_on_after_starts_on, on: :schedule
  validates :parts, presence: true, on: :parts

  # The full set, enforced once at the end and by any other writer.
  validates :title, :scheduled_on, presence: true, on: :review

  private

  def ends_on_after_starts_on
    return if starts_on.blank? || ends_on.blank?
    return if ends_on >= starts_on

    errors.add(:ends_on, "must be on or after the start date")
  end
end
```

> **The record is created before step 1, not after step 4.** A work order in progress is a real row with a `draft` state, which is why refresh, deep links, and "finish this tomorrow" all work without any extra machinery. If your schema won't tolerate partial rows, that is a schema problem worth fixing rather than a reason to hold state in the browser.

The shared step layout carries progress markup for the web and the bridge controller for native:

```erb
<%# app/views/work_orders/steps/_wizard.html.erb %>
<div data-controller="bridge--wizard"
     data-bridge--wizard-step-value="<%= step_number %>"
     data-bridge--wizard-total-value="<%= step_count %>"
     data-bridge--wizard-label-value="<%= t(".title") %>">

  <nav class="wizard-progress" data-bridge-hide-when-native aria-label="Progress">
    <p><%= t("wizard.step_of", current: step_number, total: step_count) %></p>
    <progress value="<%= step_number %>" max="<%= step_count %>"></progress>
  </nav>

  <%= yield %>
</div>
```

```erb
<%# app/views/work_orders/steps/schedule.html.erb %>
<%= render "work_orders/steps/wizard" do %>
  <%= form_with(model: @work_order,
                url: work_order_step_path(@work_order, "schedule"),
                method: :patch,
                data: { controller: "bridge--form" }) do |form| %>

    <%= render "shared/error_messages", record: @work_order %>
    <%= render "work_orders/date_range_fields", form: form %>

    <%= form.submit "Continue",
          data: { "bridge--form-target": "submit", "bridge-hide-when-native": true } %>
  <% end %>
<% end %>
```

Each step's Continue button is just Chapter 5's submit component. The wizard component does not own it — see the chrome-ownership rule in 15.6.

### CSS

```css
/* app/assets/stylesheets/bridge.css */
[data-bridge-components~="wizard"] [data-bridge-hide-when-native] {
  display: none;
}
```

The web progress bar is hidden only when the native app can actually draw progress itself. An older build without the `wizard` component keeps the web bar, which is exactly right — better a `<progress>` element than no progress at all.

## 16.4 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `connect` | `{ step: Int, total: Int, label: String }` |

No reply. The native side displays progress and nothing else.

That is a deliberately tiny contract, and it is the point of the chapter: with steps as URLs, **Back is already handled by the native navigation stack and Continue is already handled by Chapter 5.** Progress display is the only thing left that native can do better.

## 16.5 Web

```js
// app/javascript/controllers/bridge/wizard_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "wizard"
  static values = {
    step: Number,
    total: Number,
    label: { type: String, default: "" }
  }

  connect() {
    super.connect()
    if (!this.enabled) return

    this.send("connect", {
      step: this.stepValue,
      total: this.totalValue,
      label: this.labelValue
    })
  }
}
```

Eleven lines, and it fires fresh on every step because every step is a new page.

## 16.6 iOS

```swift
// ios/MyApp/Bridge/WizardComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Displays multi-step progress in the navigation bar.
final class WizardComponent: BridgeComponent {
    override class var name: String { "wizard" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .connect:
            guard let data: MessageData = message.data() else { return }
            showProgress(step: data.step, total: data.total, label: data.label)
        }
    }

    // MARK: Private

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func showProgress(step: Int, total: Int, label: String) {
        guard let viewController else { return }

        viewController.navigationItem.prompt = String(
            localized: "Step \(step) of \(total)"
        )

        if !label.isEmpty {
            viewController.title = label
        }

        // Announce the change so VoiceOver users hear where they are.
        UIAccessibility.post(
            notification: .screenChanged,
            argument: String(localized: "\(label), step \(step) of \(total)")
        )
    }
}

// MARK: Events

private extension WizardComponent {
    enum Event: String {
        case connect
    }
}

// MARK: Message data

private extension WizardComponent {
    struct MessageData: Decodable {
        let step: Int
        let total: Int
        let label: String
    }
}
```

`navigationItem.prompt` renders a small line above the title — exactly what it exists for. It also expands the navigation bar, so the layout shifts when it appears; that is native behavior and users read it as progress, not as a glitch.

## 16.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/WizardComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import android.view.View
import androidx.appcompat.widget.Toolbar
import androidx.fragment.app.Fragment
import com.google.android.material.progressindicator.LinearProgressIndicator
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.demo.R
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Displays multi-step progress in the toolbar subtitle and, when the
 * destination layout provides one, a linear progress indicator.
 */
class WizardComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private companion object {
        const val TAG = "WizardComponent"
    }

    private val fragment: Fragment
        get() = delegate.destination.fragment

    private val toolbar: Toolbar?
        get() = fragment.view?.findViewById(R.id.toolbar)

    private val progressIndicator: LinearProgressIndicator?
        get() = fragment.view?.findViewById(R.id.wizard_progress)

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> handleConnectEvent(message)
            else -> Log.w(TAG, "Unknown event for message: $message")
        }
    }

    private fun handleConnectEvent(message: Message) {
        val data = message.data<MessageData>() ?: return

        toolbar?.apply {
            subtitle = "Step ${data.step} of ${data.total}"
            if (data.label.isNotEmpty()) title = data.label
        }

        progressIndicator?.apply {
            max = data.total
            setProgressCompat(data.step, true)
            visibility = View.VISIBLE
            contentDescription = "Step ${data.step} of ${data.total}"
        }

        fragment.view?.announceForAccessibility(
            "${data.label}, step ${data.step} of ${data.total}"
        )
    }

    @Serializable
    data class MessageData(
        @SerialName("step") val step: Int,
        @SerialName("total") val total: Int,
        @SerialName("label") val label: String = ""
    )
}
```

The `LinearProgressIndicator` is looked up rather than created, and the component degrades silently when the destination's layout doesn't have one. That keeps the component usable in an app whose fragment layout you haven't modified — a property worth designing for, since bridge components are often added long after the native app's layouts were written.

## 16.8 Verify

1. Browser: the four steps navigate forward, the progress bar advances, and **browser back returns to the previous step** with its values intact.
2. Browser: submit a step with a validation error → you stay on that step, errors render, other steps are unaffected.
3. Browser: copy the step-3 URL, open it in a new tab → it loads step 3 of that record.
4. Browser: refresh mid-flow → nothing is lost, because nothing was in the browser.
5. App: the web progress bar is hidden; "Step 2 of 4" appears in the navigation bar (iOS) or toolbar subtitle (Android).
6. App: **the native back button returns to the previous step.** No bridge component is involved; this is the test that proves 16.2.
7. App: complete all four steps → the final redirect leaves the flow.
8. VoiceOver / TalkBack: moving between steps announces the new step.

Step 6 is the whole argument. If it fails, something has crept back into client-side state.

## 16.9 Fallback

`this.enabled` is false → nothing is sent and the web progress bar is never hidden. The flow is four ordinary Rails pages, which is what it is in every browser.

## 16.10 Variations

- **Skippable or branching steps** — compute `STEPS` per record rather than as a constant. The progress numbers follow automatically because they are derived, not stored.
- **A review step that edits earlier steps** — link back to each step's URL. Free, because they are URLs.
- **Saving a draft and leaving** — already works; the record exists. Add a "Finish later" link that redirects out, and an index of drafts.
- **Preventing accidental exit mid-flow** — that is Chapter 17.

## 16.11 Gotchas

1. **Native back exits the whole flow.** Steps aren't separate URLs, or a step renders with `Turbo.visit(..., action: "replace")`, which replaces the stack entry instead of pushing one.
2. **Validation errors wipe earlier steps' data.** You rebuilt the record from params instead of loading it. Load, `assign_attributes`, `save(context:)`.
3. **`save(context:)` skips validations you expected.** A validation with no `on:` runs in every context; one with `on: :details` runs *only* in that context — including not running on a plain `save`. Anything that must always hold needs a context-free validation as well, and the `on: :review` line exists for exactly that reason.
4. **The prompt sticks around after leaving the flow.** `navigationItem.prompt` belongs to the view controller, so it goes when the screen does — but if you set it on a cached or reused destination it can linger. Set it only from `connect`.
5. **Progress is wrong after a validation error.** `render @step` keeps the same step number, which is correct. If it jumps, you're deriving progress from a persisted column instead of from the current URL.

---

# Chapter 17 — The unsaved-changes guard

*Complexity 5 · ~70 lines JS, ~130 Swift, ~110 Kotlin*

The hardest component in the guide, and the one most often shipped subtly broken. It is also the clearest example of a bridge component whose job is to make a **native gesture** behave correctly with respect to web state.

## 17.1 The web problem

A half-filled form and an accidental navigation. On the web you have two hooks and both are awkward:

```js
window.addEventListener("beforeunload", (event) => {
  if (!dirty) return
  event.preventDefault()
  event.returnValue = ""   // browsers ignore your message and show their own
})

document.addEventListener("turbo:before-visit", (event) => {
  if (dirty && !window.confirm("Discard changes?")) event.preventDefault()
})
```

`beforeunload` covers closing the tab and typing a new URL; `turbo:before-visit` covers in-app links. Neither covers anything a native app does.

## 17.2 What native back actually is

In a Hotwire Native app the exits are native gestures, and **none of them produce a web event**:

| Exit | iOS | Android |
|---|---|---|
| Back button | navigation bar item | toolbar up / system back |
| Back gesture | edge swipe (`interactivePopGestureRecognizer`) | predictive back gesture |
| Modal dismissal | swipe down on a sheet | back on a dialog |

`turbo:before-visit` never fires for any of them, because there is no visit — the native stack simply pops. So the web guard is not merely inconvenient in the app; **it does not run at all.** That is what makes this a real bridge component rather than a nicety.

## 17.3 The design

The division of labor is different from every other chapter: **native owns the decision, web owns the fact.**

- The web layer knows whether the form is dirty. It tells native, and re-tells it whenever that changes.
- Native owns the back affordances. When one is triggered and the form is dirty, native asks, and pops only on confirmation.

There is no reply in the common path. Native does not need permission to pop; it needs to know whether to ask first.

## 17.4 The contract

| Direction | Event | Payload |
|---|---|---|
| Web → Native | `connect` | `{ title, message, discardTitle, keepTitle }` |
| Web → Native | `changed` | `{ dirty: Bool }` |

No replies at all. This is the guide's only component with a purely one-way contract, and it is worth noticing why: native never needs to hand anything back, because the user's answer results in a native navigation, not a web change.

## 17.5 Web

```js
// app/javascript/controllers/bridge/unsaved_changes_controller.js
import { BridgeComponent } from "@hotwired/hotwire-native-bridge"

export default class extends BridgeComponent {
  static component = "unsaved-changes"
  static values = {
    title: { type: String, default: "Discard changes?" },
    message: { type: String, default: "Your changes haven't been saved." },
    discardTitle: { type: String, default: "Discard" },
    keepTitle: { type: String, default: "Keep editing" }
  }

  connect() {
    super.connect()

    this.form = this.element.closest("form") || this.element.querySelector("form")
    if (!this.form) return

    this.snapshot = this.#serialize()
    this.dirty = false

    this.form.addEventListener("input", this.#check)
    this.form.addEventListener("change", this.#check)
    this.form.addEventListener("submit", this.#release)

    // Web guards. These never fire for native back — see 17.2.
    window.addEventListener("beforeunload", this.#beforeUnload)
    document.addEventListener("turbo:before-visit", this.#beforeVisit)

    if (this.enabled) {
      this.send("connect", {
        title: this.titleValue,
        message: this.messageValue,
        discardTitle: this.discardTitleValue,
        keepTitle: this.keepTitleValue
      })
    }
  }

  disconnect() {
    super.disconnect()
    if (!this.form) return

    this.form.removeEventListener("input", this.#check)
    this.form.removeEventListener("change", this.#check)
    this.form.removeEventListener("submit", this.#release)
    window.removeEventListener("beforeunload", this.#beforeUnload)
    document.removeEventListener("turbo:before-visit", this.#beforeVisit)

    // Leave the native app in a clean state — the next screen must not
    // inherit this screen's guard.
    if (this.enabled) this.send("changed", { dirty: false })
  }

  #check = () => {
    const dirty = this.#serialize() !== this.snapshot
    if (dirty === this.dirty) return

    this.dirty = dirty
    if (this.enabled) this.send("changed", { dirty: dirty })
  }

  // Submitting is not leaving. Stand down before the navigation starts.
  #release = () => {
    this.dirty = false
    if (this.enabled) this.send("changed", { dirty: false })
  }

  #beforeUnload = (event) => {
    if (!this.dirty) return
    event.preventDefault()
    event.returnValue = ""
  }

  #beforeVisit = (event) => {
    if (!this.dirty) return
    if (!window.confirm(this.messageValue)) event.preventDefault()
  }

  // Files are excluded deliberately: URLSearchParams stringifies a File as
  // "[object File]", so every form with a file input would compare equal no
  // matter what the user picked. If your form has one, track it separately
  // by listening for `change` on that input and setting a flag.
  #serialize() {
    const data = new FormData(this.form)
    const pairs = []

    for (const [key, value] of data.entries()) {
      if (value instanceof File) continue
      pairs.push(`${key}=${value}`)
    }

    return pairs.join("&")
  }
}
```

Two details that are easy to get wrong:

- **`#release` on submit.** Without it, submitting the form marks it clean only *after* the navigation starts, and the native guard fires on a form the user just saved. Every "why is it asking me to discard changes I just saved" bug is this line missing.
- **`send("changed", { dirty: false })` in `disconnect`.** A bridge component's native counterpart is per-screen, but the screen outlives an individual page render. Leaving without standing down means the *next* page on that screen inherits a guard for a form that no longer exists.

## 17.6 iOS

```swift
// ios/MyApp/Bridge/UnsavedChangesComponent.swift
import Foundation
import HotwireNative
import UIKit

/// Intercepts native back affordances while a web form has unsaved changes.
final class UnsavedChangesComponent: BridgeComponent {
    override class var name: String { "unsaved-changes" }

    override func onReceive(message: Message) {
        guard let event = Event(rawValue: message.event) else { return }

        switch event {
        case .connect:
            guard let data: ConnectData = message.data() else { return }
            configuration = data
        case .changed:
            guard let data: ChangedData = message.data() else { return }
            setGuard(active: data.dirty)
        }
    }

    // MARK: Private

    private var configuration: ConnectData?
    private var isGuarding = false

    private var viewController: UIViewController? {
        delegate?.destination as? UIViewController
    }

    private func setGuard(active: Bool) {
        guard active != isGuarding, let viewController else { return }
        isGuarding = active

        if active {
            installCustomBackButton(on: viewController)
        } else {
            restoreSystemBackButton(on: viewController)
        }

        // The edge-swipe gesture cannot be intercepted, only disabled.
        // Order matters: the custom back button is cleared first (above), because
        // UIKit disables the gesture on its own whenever a custom leftBarButtonItem
        // is set, and re-enabling it while one is still installed is what produces
        // the classic "swipe pops the screen but the navigation bar doesn't update"
        // hang.
        viewController.navigationController?
            .interactivePopGestureRecognizer?.isEnabled = !active

        // Sheets presented over this screen must not swipe away either.
        viewController.isModalInPresentation = active
    }

    private func installCustomBackButton(on viewController: UIViewController) {
        let action = UIAction { [unowned self] _ in askThenPop() }

        let item = UIBarButtonItem(
            image: UIImage(systemName: "chevron.backward"),
            primaryAction: action
        )
        item.accessibilityLabel = String(localized: "Back")

        viewController.navigationItem.hidesBackButton = true
        viewController.navigationItem.leftBarButtonItem = item
    }

    private func restoreSystemBackButton(on viewController: UIViewController) {
        viewController.navigationItem.leftBarButtonItem = nil
        viewController.navigationItem.hidesBackButton = false
    }

    private func askThenPop() {
        guard let viewController, let configuration else { return }

        let alert = UIAlertController(
            title: configuration.title,
            message: configuration.message,
            preferredStyle: .alert
        )

        alert.addAction(UIAlertAction(title: configuration.keepTitle, style: .cancel))

        alert.addAction(
            UIAlertAction(title: configuration.discardTitle, style: .destructive) { [unowned self] _ in
                setGuard(active: false)
                pop()
            }
        )

        viewController.present(alert, animated: true)
    }

    private func pop() {
        guard let viewController else { return }

        if let navigationController = viewController.navigationController,
           navigationController.viewControllers.count > 1 {
            navigationController.popViewController(animated: true)
        } else {
            viewController.dismiss(animated: true)
        }
    }
}

// MARK: Events

private extension UnsavedChangesComponent {
    enum Event: String {
        case connect
        case changed
    }
}

// MARK: Message data

private extension UnsavedChangesComponent {
    struct ConnectData: Decodable {
        let title: String
        let message: String
        let discardTitle: String
        let keepTitle: String
    }

    struct ChangedData: Decodable {
        let dirty: Bool
    }
}
```

> **The edge-swipe gesture is disabled, not intercepted, and that is a real cost.**
>
> `UINavigationController` gives you no hook that can conditionally cancel an interactive pop once it has begun. `UINavigationControllerDelegate` tells you a pop *happened*; `interactivePopGestureRecognizer.delegate` can refuse to *begin* — which is all-or-nothing for the screen.
>
> So the honest implementation turns the gesture off while the form is dirty and back on when it isn't. A user who has typed nothing keeps the gesture; a user mid-edit temporarily loses it and must use the back button. That is the least-bad option available through public API, and pretending otherwise produces a guard that the swipe silently walks straight through.
>
> `isModalInPresentation = true` covers the other half — a screen presented as a sheet, where the exit is a downward swipe rather than a pop.

## 17.7 Android

```kotlin
// android/app/src/main/kotlin/dev/hotwire/demo/bridge/UnsavedChangesComponent.kt
package dev.hotwire.demo.bridge

import android.util.Log
import androidx.activity.OnBackPressedCallback
import androidx.fragment.app.Fragment
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import dev.hotwire.core.bridge.BridgeComponent
import dev.hotwire.core.bridge.BridgeDelegate
import dev.hotwire.core.bridge.Message
import dev.hotwire.navigation.destinations.HotwireDestination
import kotlinx.serialization.SerialName
import kotlinx.serialization.Serializable

/**
 * Intercepts the system back gesture while a web form has unsaved changes.
 */
class UnsavedChangesComponent(
    name: String,
    private val delegate: BridgeDelegate<HotwireDestination>
) : BridgeComponent<HotwireDestination>(name, delegate) {

    private companion object {
        const val TAG = "UnsavedChangesComponent"
    }

    private var configuration: ConnectData? = null
    private var callback: OnBackPressedCallback? = null

    private val fragment: Fragment
        get() = delegate.destination.fragment

    override fun onReceive(message: Message) {
        when (message.event) {
            "connect" -> {
                configuration = message.data<ConnectData>()
                installCallback()
            }
            "changed" -> {
                val data = message.data<ChangedData>() ?: return
                callback?.isEnabled = data.dirty
            }
            else -> Log.w(TAG, "Unknown event for message: $message")
        }
    }

    private fun installCallback() {
        // viewLifecycleOwner throws if the view is already destroyed, which can
        // happen if a late message arrives during teardown. Bail rather than crash.
        val owner = runCatching { fragment.viewLifecycleOwner }.getOrNull() ?: return

        // Replace rather than stack — connect can fire again on re-render.
        callback?.remove()

        val newCallback = object : OnBackPressedCallback(false) {
            override fun handleOnBackPressed() = askThenPop()
        }

        fragment.requireActivity()
            .onBackPressedDispatcher
            .addCallback(owner, newCallback)

        callback = newCallback
    }

    private fun askThenPop() {
        val context = fragment.context ?: return
        val configuration = configuration ?: return

        MaterialAlertDialogBuilder(context)
            .setTitle(configuration.title)
            .setMessage(configuration.message)
            .setCancelable(true)
            .setNegativeButton(configuration.keepTitle, null)
            .setPositiveButton(configuration.discardTitle) { _, _ -> pop() }
            .show()
    }

    private fun pop() {
        // Stand down first, or the dispatcher hands the event straight back.
        callback?.isEnabled = false
        fragment.requireActivity().onBackPressedDispatcher.onBackPressed()
    }

    @Serializable
    data class ConnectData(
        @SerialName("title") val title: String,
        @SerialName("message") val message: String,
        @SerialName("discardTitle") val discardTitle: String,
        @SerialName("keepTitle") val keepTitle: String
    )

    @Serializable
    data class ChangedData(
        @SerialName("dirty") val dirty: Boolean
    )
}
```

Android comes out ahead here. `OnBackPressedCallback` is a first-class interception point that covers the button *and* the gesture, it is lifecycle-aware so it goes away with the view automatically, and toggling `isEnabled` is exactly the "guard on / guard off" primitive this component needs. There is no equivalent on iOS, which is why 17.6 is longer and lossier.

Two ordering details matter:

- **`callback?.isEnabled = false` before `onBackPressed()`.** An enabled callback receives its own dispatched event and loops forever.
- **`viewLifecycleOwner`, not `fragment`.** Tying the callback to the fragment rather than its view leaves it registered across view recreation, and you accumulate one guard per rotation.

## 17.8 Rails

```erb
<%# app/views/work_orders/edit.html.erb %>
<div data-controller="bridge--unsaved-changes"
     data-bridge--unsaved-changes-title-value="<%= t(".discard_title") %>"
     data-bridge--unsaved-changes-message-value="<%= t(".discard_message") %>"
     data-bridge--unsaved-changes-discard-title-value="<%= t(".discard") %>"
     data-bridge--unsaved-changes-keep-title-value="<%= t(".keep_editing") %>">

  <%= render "form", work_order: @work_order %>
</div>
```

No variant, no CSS, no server involvement. The component wraps whatever form is already there.

## 17.9 Verify

Each of these is a separate failure mode and each has shipped broken somewhere:

1. Browser: edit a field, click a link → confirmation. Cancel → you stay, with your edit.
2. Browser: edit a field, close the tab → the browser's own warning.
3. Browser: edit, then **change it back to the original value** → no warning. This is what snapshot comparison buys over a boolean flag.
4. Browser: submit the form → no warning during the redirect.
5. App: edit a field, tap native back → native alert. "Keep editing" → you stay.
6. App: "Discard" → the screen pops.
7. **App, iOS: edit a field, then edge-swipe** → nothing happens, because the gesture is disabled. Clear the field back to its original value → the gesture works again.
8. **App, Android: edit a field, then use the system back gesture** → the alert appears. This is the case `beforeunload` can never cover.
9. App: submit successfully, land on the next screen, tap back → **no alert.** This is the `#release` + `disconnect` test and it is the most commonly broken one.
10. App: navigate to this screen, leave without editing, come back, edit, back → the alert still works. Proves the guard is re-armed rather than one-shot.
11. **App, Android: rotate the device three times while dirty, then tap back** → exactly one alert, not three. Proves `viewLifecycleOwner` and the `callback?.remove()`.

## 17.10 Fallback

`this.enabled` is false → no messages are sent and the component is nothing but the two web listeners, which is the standard web guard. Every browser gets the behavior it always had.

## 17.11 Variations

- **Autosave instead of guarding** — the better product answer where it fits. Debounce a `PATCH` on `input` and never ask the question. Keep this component installed anyway, guarding only the window between the last keystroke and the save landing.
- **Scoping the snapshot** — comparing the whole form is noisy if something else mutates hidden fields. Serialize only inputs matching a selector.
- **Per-field dirty indicators** — a web concern; don't send it over the bridge.
- **iOS: keeping the swipe gesture** — possible with a custom interactive transition that can be cancelled, at a cost far above this component's value. The disable-while-dirty approach is the recommended one.

## 17.12 Gotchas

1. **Asked to discard changes right after saving.** `#release` isn't firing on `submit`, or it fires after the visit starts. Bind it on the form's `submit` event, not on the button's `click`.
2. **The guard follows you to the next screen.** No `changed: false` on `disconnect`.
3. **Android back does nothing at all.** `callback?.isEnabled` stayed `true` in `pop()`, so the dispatcher re-delivered the event to the same callback.
4. **Multiple alerts stack on Android.** The callback was registered per `connect` without `remove()`, or it was tied to the fragment instead of `viewLifecycleOwner`.
5. **The iOS edge swipe walks past the guard.** `interactivePopGestureRecognizer.isEnabled` wasn't toggled. There is no way to intercept it mid-gesture; see 17.6.
6. **A sheet swipes away over a dirty form.** `isModalInPresentation` wasn't set.
7. **Dirty on load with no user input.** Something mutates the form after `connect` — a date component writing a normalized value, a `<select>` defaulting. Take the snapshot in `requestAnimationFrame`, after other controllers have settled.

8. **The guard survives a tab switch.** `disconnect()` has been reported not to fire when the user switches tabs (20.4 entry 9), so the stand-down message in 17.5 never sends. This component is unusually exposed to it, because a guard that outlives its form is worse than no guard. Do not rely on `disconnect()` as the only teardown path: have the native side also clear its guard when its destination goes away, so cleanup has two independent triggers.

Gotcha 7 is worth expanding, because it is the one that makes users distrust the feature: a guard that fires when nothing was edited trains people to tap "Discard" reflexively, and then it fails the one time it mattered.

---

# Chapter 18 — Testing across three codebases

A bridge component can break in three places, and two of them are invisible from the third.

## 18.1 The layers

| Test | Proves | Runs in |
|---|---|---|
| Rails system test | The web fallback works — the path most users are on | CI, every commit |
| Contract test | All three sides agree on the payload shape | CI, every commit |
| XCTest / Espresso | The native component behaves | CI, per platform |
| Manual device pass | The whole thing feels right | Before release |

## 18.2 Always test the fallback in Rails

Every component chapter's system test runs with **no bridge at all**. That is deliberate: it pins the behavior your desktop users, mobile-browser users, and out-of-date-app users get. If the only tests you have require a simulator, you have no coverage of the majority path.

## 18.3 The contract test

The highest-value test you can add, and the cheapest. Define each component's payload shape once:

```ruby
# test/bridge/contracts/date_picker.json
{
  "component": "date-picker",
  "events": {
    "display": {
      "send": {
        "title": "String",
        "value": "String?",
        "min": "String?",
        "max": "String?"
      },
      "reply": {
        "value": "String"
      }
    }
  }
}
```

Then assert, in each codebase's own test suite, that its structs match. On iOS that is a round-trip test:

```swift
// ios/MyApp/Bridge/DatePickerPayloads.swift
// Lift the payload types out of the component's private extension so
// tests can reach them. The component keeps using them unchanged.
import Foundation

enum DatePickerPayloads {
    struct Message: Decodable {
        let title: String
        let value: String?
        let min: String?
        let max: String?
    }

    struct Selection: Encodable {
        let value: String
    }
}
```

```swift
// ios/MyAppTests/DatePickerContractTests.swift
import XCTest
@testable import MyApp

final class DatePickerContractTests: XCTestCase {
    func testIncomingPayloadMatchesContract() throws {
        let json = """
        {
          "title": "Scheduled date",
          "value": "2026-03-14",
          "min": "2026-03-12",
          "max": "2027-03-12"
        }
        """

        let decoded = try JSONDecoder().decode(
            DatePickerPayloads.Message.self,
            from: Data(json.utf8)
        )

        XCTAssertEqual(decoded.value, "2026-03-14")
        XCTAssertEqual(decoded.min, "2026-03-12")
    }

    func testReplyPayloadMatchesContract() throws {
        let data = try JSONEncoder().encode(
            DatePickerPayloads.Selection(value: "2026-03-19")
        )

        let object = try JSONSerialization.jsonObject(with: data) as? [String: Any]

        XCTAssertEqual(object?["value"] as? String, "2026-03-19")
    }
}
```

Note what the first test buys that the second doesn't: `message.data()` returns an **optional** and swallows decode failures silently (A.2). A renamed key produces `nil` at runtime and no log line. Decoding the contract's own example JSON in a test is the only place that failure becomes loud.

The failure mode this catches is the expensive one: the web team renames `value` to `date`, iOS ships in the next release, Android doesn't, and the bug is only visible on one platform in production.

## 18.4 Test the timezone round-trip, not just the happy path

Any component carrying a date or time gets tests at UTC+13 and UTC−10, on both platforms. Assume nothing about the CI machine's timezone.

---

# Chapter 19 — Debugging

## 19.1 "Nothing happens"

Work down this list; it is ordered by how often each cause is the real one.

**1. Name mismatch.** Check all three literally, character by character:

```js
static component = "date-picker"     // JS
```
```swift
override class var name: String { "date-picker" }   // iOS
```
```kotlin
BridgeComponentFactory("date-picker", ::DatePickerComponent)   // Android
```

A common variant is Stimulus-identifier confusion: `bridge--date-picker` in `data-controller` is correct, but `static component` must be `"date-picker"`, not `"bridge--date-picker"`.

**2. Registration ran too late.** On iOS the registration list is baked into the WebView's user agent, and that string is built **when the `Navigator`'s `rootViewController` is first accessed** — not when the first visit happens. Registering after that point is silently too late. Put `Hotwire.registerBridgeComponents` at the top of `didFinishLaunchingWithOptions`, before any `Navigator` or `Session` is constructed *or touched*. On Android, the top of `Application.onCreate`. See 20.2 entry 1.

**2b. The bridge library was imported too late.** Importing `@hotwired/hotwire-native-bridge` is what creates the bridge object; controllers loaded before it have nothing to attach to. It goes above `import "controllers"` in `application.js`. See 20.2 entry 2.

**3. The component isn't advertised.** In the WebView inspector:

```js
document.documentElement.dataset.bridgeComponents
// → "form toast confirm select date-picker multi-select date-range typeahead nested-form wizard unsaved-changes"
```

If your component isn't in that string, the native app didn't register it — or you're running an older build.

**4. `super.connect()` missing.** If you overrode `connect()` without calling `super.connect()`, the component never registers. This produces exactly the same silence as a name mismatch.

**5. The Stimulus controller is registered twice.** Stimulus silently refuses to connect a controller registered more than once — indistinguishable from a controller that was never registered. Check the live registry:

```js
window.Stimulus.router.modules.map((m) => m.identifier).sort()
```

Look for a duplicate identifier, or for `bridge--date-picker` sitting alongside a bare `date-picker`.

**6. iOS only: the Swift file isn't in the build target.** The project builds, the file exists, and the class does not. Select it in Xcode and check **Target Membership** in the File Inspector. Check this early — it costs five seconds.

**7. You are testing in a desktop browser.** Bridge components never connect in a browser; `this.enabled` is `false` by design. Component work has to be verified in a simulator or on a device.

**8. Check `this.enabled` directly.**

```js
connect() {
  super.connect()
  console.log(this.component, "enabled:", this.enabled)
}
```

## 19.2 Inspecting traffic

**iOS** — Safari → Develop → your device → the WebView. Standard Web Inspector, including the console and `document.documentElement.dataset`.

**Android** — `chrome://inspect` in desktop Chrome with USB debugging on. Requires `WebView.setWebContentsDebuggingEnabled(true)` in debug builds.

Log every message during development:

```swift
override func onReceive(message: Message) {
    #if DEBUG
    print("[\(Self.name)] ← \(message.event): \(message.jsonData)")
    #endif
    // … handling
}
```

```kotlin
override fun onReceive(message: Message) {
    if (BuildConfig.DEBUG) Log.d(TAG, "← ${message.event}: $message")
    // … handling
}
```

## 19.3 "The reply never arrives"

The callback in `send` fires only when the native side calls `reply(to:)` / `replyTo()` with **the same event name**. Replying to `"connect"` will not fire a callback registered by `send("display", …)`. Spelling and case must match.

## 19.4 "It works on one screen and not another"

`delegate?.destination` is per-screen. A component holding a reference to a view controller or fragment from a previous screen will silently target the wrong one, or nil. Re-read `delegate?.destination` on each use rather than caching it in a property.

---

# Chapter 20 — Field notes: upstream issues worth knowing

*Read once; return when something inexplicable happens*

Everything in this chapter comes from a public bug report, a maintainer's diagnosis, or a thorough answer on the Hotwire forum. Each entry is something that cost a real team real time, and most of them share a property that makes them expensive: **the symptom points somewhere other than the cause.**

## 20.1 How to read this chapter

> **This chapter ages faster than the rest of the guide.** Every entry carries its source and the version it was reported against. Before acting on one, open the link and check whether it has been fixed — several of these are open issues that may well be closed by the time you read this.

Entries are grouped by what you *observe*, not by what is wrong, because the observation is all you have when you start. Each one is: **symptom → why it's hard → what's actually happening → what to do**.

Nine of the twelve are things a bridge-component author hits specifically. The other three are framework-level behaviors that will be blamed on your component.

---

## 20.2 Nothing happens at all

This is the signature failure of bridge components, and it has at least four distinct causes that produce an identical symptom: no error, no warning, no log line, no native UI.

### 1 · The component was registered after the user agent was built

**Symptom.** Everything looks correct. The Stimulus controller is loaded, the Swift class compiles, the names match — and `this.enabled` is `false`.

**Why it's hard.** There is nothing to see. The bridge's registration list is baked into the WebView's user agent string, and that string is built once.

**What's actually happening.** On iOS the user agent is established **when the `Navigator`'s `rootViewController` is first accessed**. Registering components after that point is too late; the list has already been sent. A forum thread traced a complete non-working setup to documentation that accessed `rootViewController` before the registration call.

**What to do.** Register in `didFinishLaunchingWithOptions`, before any `Navigator` or `Session` is constructed or touched. Then verify from the web side rather than trusting the code:

```js
document.documentElement.dataset.bridgeComponents
```

If your component name isn't in that string, nothing downstream can work. Source: [Hotwire forum #6024](https://discuss.hotwired.dev/t/unable-to-get-a-native-bridge-working-with-rails-7-2-and-hotwire-native-bridge-1-0-0/6024).

### 2 · The bridge library was imported too late

**Symptom.** Same as above, but from the web side: the Stimulus controller's `initialize` and `connect` never fire, while an ordinary Stimulus controller on the same element works fine.

**What's actually happening.** Importing `@hotwired/hotwire-native-bridge` is what creates the bridge object. If it is imported after the controllers that extend `BridgeComponent`, those controllers have nothing to attach to.

**What to do.** Import it at the top of `app/javascript/application.js`, before your controllers:

```js
// app/javascript/application.js
import "@hotwired/turbo-rails"
import "@hotwired/hotwire-native-bridge"
import "controllers"
```

Same source as entry 1. The diagnostic that isolated it is worth copying: **swap `BridgeComponent` for a plain `Controller` and see whether lifecycle events fire.** If they do, the problem is the bridge, not Stimulus.

### 3 · The Stimulus controller is registered twice

**Symptom.** The controller silently never connects. No error.

**Why it's hard.** Stimulus does not warn about this. A duplicate registration is a no-op that looks exactly like a missing one.

**What's actually happening.** Eager-loading conventions can register a controller under more than one identifier — commonly when bridge controllers live in a subdirectory *and* are also registered manually, or when two loading mechanisms are both active.

**What to do.** Keep bridge controllers in exactly one place with exactly one loading mechanism, and check the registrations at runtime:

```js
// In the WebView console
window.Stimulus.router.modules.map((m) => m.identifier).sort()
```

Look for the same identifier twice, or for an identifier you didn't expect — `bridge--date-picker` registered alongside a bare `date-picker` is the usual shape. Source: [Jesse Waites — Debugging custom Bridge Components](https://jessewaites.com/blog/post/debugging-custom-bridge-components-in-hotwire-native/).

### 4 · The Swift file isn't in the build target

**Symptom.** The component is registered in `AppDelegate`, the file exists on disk, the project builds — and the component doesn't exist at runtime.

**Why it's hard.** It builds. There is no compiler error, because Xcode does not compile files it does not believe belong to a target, even when they are inside the project directory.

**What to do.** Select the file in Xcode and check **Target Membership** in the File Inspector. Do this before debugging anything else on iOS; it costs five seconds and it is a surprisingly common cause. Same source as entry 3.

> **A fifth cause, and the one that catches everyone once: you are testing in a desktop browser.** Bridge components do not connect in a browser at all — that is the entire point of `this.enabled`, and Chapter 4 is built on it. Component work has to be verified in a simulator or on a device.

---

## 20.3 The page shows the wrong thing

### 5 · A cached snapshot hides your flash message or validation errors

**Symptom.** A redirect lands on a URL already in the navigation stack and the user sees the *old* content — no flash, no error, no updated CSRF token.

**What's actually happening.** Hotwire Native restores the cached snapshot for that URL in preference to the server's fresh response. Anything session-dependent that the server just generated is lost.

**Why this matters here.** Chapter 6's toast component sends `display` from `connect()`. If the flash partial comes from a snapshot, that message may be a *stale* one — or absent entirely. The component is working correctly and showing you the wrong data.

**What to do.** Avoid redirecting to a URL already on the stack when the response carries session-dependent content; prefer a distinct URL or a Turbo Stream. Track the issue for a server-driven cache-control header. Source: [hotwire-native-ios#178](https://github.com/hotwired/hotwire-native-ios/issues/178), opened September 2025.

### 6 · The flash is consumed before your toast can show it

**Symptom, Android.** A form POSTs, redirects, the page renders — and the flash message is missing. It works in a browser and on iOS.

**Why it's hard.** Nothing in your component is wrong, and the Rails log looks almost normal until you count the requests.

**What's actually happening.** After a POST, the Android client issues the follow-up GET **twice** — once as `TURBO_STREAM` and once as `HTML`. The first request consumes the flash; the second renders without it. iOS issues only the `TURBO_STREAM` request, which is why it works there — and iOS reportedly exhibits the same double request when `context: modal` is removed from the path configuration.

**What to do.** Count requests in the Rails log after a redirect before blaming the toast component. Where it bites, `flash.keep` on the follow-up render, or delivering the message as a Turbo Stream rather than a flash, both sidestep it. Source: [hotwire-native-android#198](https://github.com/hotwired/hotwire-native-android/issues/198).

### 7 · A tracked-asset mismatch freezes the screen on back navigation

**Symptom.** Navigating back leaves a loading spinner that never goes away. The app is not crashed; it is stuck.

**Why it's hard.** It reproduces only after an asset fingerprint changes, so it often appears first in production or after a deploy mid-session — and it looks like a navigation bug, not an asset bug.

**What's actually happening.** A maintainer-quality diagnosis in the thread: `pageInvalidated` fires during a **restore** visit, which triggers `cancelVisit` followed by a `ColdBootVisit`, and in that transition the loading overlay is never dismissed. The trigger is pages whose `data-turbo-track="reload"` asset lists differ from one another.

**What to do.** **Tracked assets must be invariant across every navigable page.** Keep `data-turbo-track="reload"` on truly global bundles only — `application.css`, `application.js` — and never on page-specific assets.

> This one has a direct bearing on Chapter 3. A **variant layout is exactly where asset lists diverge**, because it is a second layout maintained separately. If `layouts/hotwire_native.html.erb` tracks a different set of assets than `layouts/application.html.erb`, you have built the precondition for this bug. Keep the tracked set identical in both, or track nothing in the native layout.

Source: [hotwire-native-ios#218](https://github.com/hotwired/hotwire-native-ios/issues/218).

---

## 20.4 Navigation and lifecycle

### 8 · A form redirect pops you back to the form

**Symptom.** Submit a form, the record is created — and you land back on the form screen instead of the record.

**What's actually happening.** `Navigator.session(_:didProposeVisit:)` called `pop(animated:)` whenever `proposal.isRedirect` was true. That was meant for modal dismissal, but it keyed only off `options.response?.redirected == true`, so it fired for redirects on the main stack too. The pop reveals the underlying screen, whose `viewWillAppear` fires a restore visit, which cancels the in-flight redirect and discards its HTML. Turbo then re-renders the form from its snapshot cache.

**Why it matters here.** This is the exact path Chapter 5's submit button and Chapter 16's wizard steps take on every submission: POST → 303 → push. If it is broken, both chapters appear broken.

**What to do.** Reported against **1.3.0-beta**; 1.2.2 and Android were unaffected. Pin a version you have tested this path on, and make "submit a form on the main stack and confirm you land on the record" part of your upgrade checklist. Source: [hotwire-native-ios#244](https://github.com/hotwired/hotwire-native-ios/issues/244).

### 9 · `disconnect()` is not called on tab switch

**Symptom.** A component that changes global state — screen brightness, an audio session, a guard — does not clean up when the user switches tabs.

**Why it matters here.** **Chapter 17 depends on `disconnect()`**, which sends `changed: { dirty: false }` so the next screen does not inherit the guard. If `disconnect()` does not fire on a tab switch, a dirty form on one tab can leave a back-guard armed on another.

**What to do.** Do not rely on `disconnect()` as your only teardown path for anything that affects state outside the component. Have the native side also stand down when its destination goes away, so cleanup has two independent triggers rather than one. Source: [hotwire-native-bridge#10](https://github.com/hotwired/hotwire-native-bridge/issues/10).

### 10 · The WebView has no focus after navigation

**Symptom, Android.** Document-level keyboard listeners — `keydown` on `document` — stop firing after a navigation, until the user touches the screen.

**Why it's hard.** It works in a browser, works on the first page, and "fixes itself" the moment you tap to investigate. Anyone debugging by hand will struggle to reproduce it.

**What's actually happening.** The WebView does not hold focus after navigation. A desktop browser keeps focus on the document; the WebView does not, so nothing receives key events until a touch grants it.

**What matters practically.** Barcode scanners and other HID-style input devices present as keyboards. On Android they go silent after every navigation.

**What to do.** Request focus when the WebView attaches:

```kotlin
override fun onWebViewAttached(webView: HotwireWebView) {
    webView.requestFocus()
}
```

Source: [hotwire-native-android#124](https://github.com/hotwired/hotwire-native-android/issues/124), with the workaround from the reporter.

---

## 20.5 Dialogs and permissions

### 11 · A dropped JavaScript alert completion handler crashes the app

**Symptom.** `NSInternalInconsistencyException` on the next WebView interaction, some time after a dialog disappeared. The crash is nowhere near the cause.

**What's actually happening.** `WKUIController.runJavaScriptConfirmPanelWithMessage` (and the alert variant) presents a `UIAlertController` and calls `completionHandler(_:)` **only from inside the `UIAlertAction` closures**. If the presenting view controller is torn down externally — a deep link, a push-notification handler, a programmatic dismissal — the alert goes away without any action firing, the handler is never called, and WebKit raises on the next interaction.

**Why it matters here.** Chapter 7 replaces this code path, so a confirm bridge component sidesteps the framework bug. But the underlying lesson is a rule for *your* components: **a reply path that can be skipped is a bug, not an edge case.** Chapter 7's Android `setOnCancelListener` exists for exactly this reason — and entry 11 shows the iOS side is not as immune as "a `UIAlertController` can't be dismissed without choosing an action" suggests. It can, if something dismisses its presenter.

**What to do.** For any component holding a pending web callback, reply from teardown as well as from the buttons. Source: [hotwire-native-ios#239](https://github.com/hotwired/hotwire-native-ios/issues/239).

### 12 · The Android WebView never asks for camera or location permission

**Symptom.** `navigator.geolocation`, `getUserMedia`, and camera-backed `<input type="file" capture>` fail **silently** in the Android app. The same page works in Chrome on the same device.

**Why it's hard.** There is no permission prompt, no console error, and no exception — the feature simply does nothing.

**What's actually happening.** The Android WebView does not request runtime permissions on its own. Without host-side handling of the relevant WebChromeClient callbacks and the corresponding manifest permissions, the web API is denied before the user ever sees a prompt.

**Why it matters here.** This is the single most important input to Appendix B. It means **"just use the web API" is not a viable fallback on Android** for camera, location, or media capture — which removes an option that would otherwise make several of the deferred components unnecessary. Any chapter covering those has to treat native permission handling as mandatory rather than as an enhancement.

**What to do.** Treat device capabilities on Android as requiring native work. Verify the current state of this before designing around it. Source: [hotwire-native-android#134](https://github.com/hotwired/hotwire-native-android/issues/134).

---

## 20.6 The pattern across all twelve

Read together, these entries have a shape:

| Class | Entries | What they have in common |
|---|---|---|
| **Ordering** | 1, 2, 3, 4 | Something correct happened at the wrong time, or in the wrong build. No error is produced because nothing failed — it just never ran. |
| **Caching** | 5, 6, 7 | The page you are looking at is not the page the server sent. Your component is faithfully rendering stale input. |
| **Lifecycle** | 8, 9, 10 | A native transition happened that the web layer never learned about. |
| **Unclosed paths** | 11, 12 | Something was promised — a completion handler, a permission prompt — and never delivered. Silence, not failure. |

Three habits follow, and they cover most of this chapter:

1. **Verify registration from the web side, not the code.** `document.documentElement.dataset.bridgeComponents` is the ground truth for the ordering class. Code that looks right proves nothing.
2. **Count the requests in the Rails log.** The caching class shows up there before it shows up anywhere else — a duplicate GET, a request you didn't expect, a missing one.
3. **Make every waiting path terminate.** Every `send` with a callback, every completion handler, every permission request. Ask "what if this never comes back?" — and then make sure it always does.

---

# Chapter 21 — Should this be a bridge component?

## 21.1 The cost, honestly

Every bridge component costs:

- three implementations that must be changed together
- an App Store / Play Store review cycle on any native change
- a version-skew matrix — some users will run the old native side against the new web side
- a fallback path that must be maintained and tested regardless

That is a real ongoing cost. It is worth paying when the payoff is large, and it is a bad trade when it isn't.

## 21.2 The decision path

This is question 1 from 3.2 — *what mechanism does the behavior need?* Delivery is a separate decision and comes after; the second flowchart covers it.

```
Is the interaction possible in plain HTML with the right attributes?
│  (inputmode, autocomplete, enterkeyhint, type)
├── YES → Do that. Chapter 8. Stop.
└── NO
    │
    Is the difference purely that the native app needs different
    markup — less chrome, a different arrangement, fewer elements?
    ├── YES → A variant partial is the whole answer. Chapter 3. Stop.
    └── NO — it needs a native control or a device capability
        │
        Does the native platform have a first-party control for this?
        ├── NO → Keep it on the web. A hand-built native control has all
        │        the cost of a bridge component and none of the payoff.
        │        Chapters 13 and 16 are what this branch looks like.
        └── YES
            │
            Would the native version need to own validation or state
            that Rails already owns?
            ├── YES → Redesign so it doesn't. If you can't, don't bridge it.
            └── NO
                │
                Is the web version >150 lines of JS, or a dependency,
                or a recurring source of mobile bugs?
                ├── NO  → Not yet. Revisit when it is.
                └── YES → Bridge it.
```

Then, and only then, question 2 — *how does its markup reach the page?*

```
Does the native path need elements a browser has no use for,
or does the browser load an enhancement the app shouldn't?
├── NO  → Inline it in the shared partial. One template.
│         Chapters 5, 6, 7, 9, 10, 12, 15, 16, 17.
└── YES → Variant partial. Chapters 11 and 14 in practice.
          Either way: keep `this.enabled` and the CSS rule.
          The server knows browser-or-app; only the client
          knows which components this app actually has.
```

## 21.3 Applied to this guide's components

| Component | Native control exists | Web JS replaced | Verdict |
|---|---|---|---|
| Submit button | ✅ | ~120 lines of sticky-footer JS | **Bridge** |
| Flash toast | ✅ | ~80 lines + timers | **Bridge** |
| Confirm dialog | ✅ | ~150 lines modal + focus trap | **Bridge** |
| Numeric keyboard | ✅ | 0 — `inputmode` does it | **Don't** |
| OTP field | ✅ | 0 — `autocomplete` does it | **Don't** |
| Select | ✅ | 40–70 KB library | **Bridge** |
| Date picker | ✅ | 40–60 KB + timezone bugs | **Bridge** |
| Multi-select | ✅ | 40–70 KB library | **Bridge** |
| Date range | ✅ Android, ❌ iOS | dual-calendar widget | **Bridge, asymmetric** |
| Remote typeahead | ✅ | debounce + races + cache | **Bridge** |
| Dependent selects | ❌ | ~80 lines of chained fetch | **Turbo Frames instead** |
| Nested fieldset rows | ❌ | cocoon or equivalent | **Web; native affordances only** |
| Multi-step wizard | ❌ | a client state machine | **Server-driven URLs; native shows progress** |
| Unsaved-changes guard | ✅ (the gesture) | `beforeunload`, which native never fires | **Bridge — nothing else works** |
| Masked currency | ⚠️ partial | ~100 lines | **Probably not** |
| Rich text editor | ❌ (no native Trix) | — | **Toolbar only** |

Four rows say **don't**, and they are the ones worth reading twice. "Dependent selects" and "multi-step wizard" both have no native control at all, so the answer is a better server-side design (Chapters 13 and 16). "Nested fieldset rows" has no native control for the rows themselves, so only the surrounding affordances move (Chapter 15). And "unsaved-changes guard" is the opposite case — the only row where a bridge component isn't an improvement but the *sole* option, because the native back gesture produces no web event at all.

---

# Appendix A — API reference

The API surface this guide was written against, at the versions in the front matter. Confirm anything load-bearing against the version you have installed.

## A.1 Web — `@hotwired/hotwire-native-bridge` 1.2.2

```
class BridgeComponent extends Controller {
  static component = ""          // must match the native component name

  // Stimulus lifecycle — call super when overriding
  initialize()
  connect()
  disconnect()

  get component()                // the static component name
  get platformOptingOut()        // data-controller-optout-ios / -android
  get enabled()                  // is this component registered natively?
  get bridgeElement()            // wraps this.element as a BridgeElement
  get bridge()

  send(event, data = {}, callback)
}

class BridgeElement {
  constructor(element)
  get title()                    // data-bridge-title → aria-label → textContent
  get enabled()
  get disabled()                 // data-bridge-disabled
  enableForComponent(component)
  hasClass(className)
  attribute(name)
  bridgeAttribute(name)          // reads data-bridge-<name>
  setBridgeAttribute(name, value)
  removeBridgeAttribute(name)
  click()
  get platform()
}
```

The callback passed to `send` receives a **message object**; the payload is at `message.data`.

## A.2 iOS — `hotwire-native-ios` 1.3.0

```swift
class BridgeComponent {
    override class var name: String { get }      // must be overridden

    func onReceive(message: Message)             // must be overridden
    func reply(to event: String)                 // reply with no data
    func reply(to event: String, with data: some Encodable)

    var delegate: BridgeDelegate? { get }
}

struct Message {
    let event: String
    func data<T: Decodable>() -> T?              // returns nil on decode failure
}
```

Registration: `Hotwire.registerBridgeComponents([MyComponent.self])`
Screen access: `delegate?.destination as? UIViewController`

> `message.data()` returns an **optional** and swallows decode errors. A typo in a payload key produces `nil`, not a crash and not a log line. `guard let data: MessageData = message.data() else { return }` is silent failure — add a debug log to that `else` branch during development.

## A.3 Android — `hotwire-native-android` 1.3.1

```kotlin
abstract class BridgeComponent<D : BridgeDestination>(
    name: String,
    delegate: BridgeDelegate<D>
) {
    abstract fun onReceive(message: Message)
    fun replyTo(event: String): Boolean
    fun <T> replyTo(event: String, data: T): Boolean   // T must be @Serializable
}

class Message {
    val event: String
    inline fun <reified T> data(): T?
}
```

Registration: `Hotwire.registerBridgeComponents(BridgeComponentFactory("name", ::MyComponent))`
Screen access: `delegate.destination.fragment`

Generic parameter is `HotwireDestination`, imported from `dev.hotwire.navigation.destinations.HotwireDestination`. Older material referring to `BridgeDestination` as the type argument is out of date.

## A.4 Rails — `turbo-rails`

```ruby
# Turbo::Native::Navigation, mixed into ActionController::Base
# Available as both a controller method and a view helper.

hotwire_native_app?          # request.user_agent.to_s.match?(/(Turbo|Hotwire) Native/)
turbo_native_app?            # alias of the above

recede_or_redirect_to(url, **options)
resume_or_redirect_to(url, **options)
refresh_or_redirect_to(url, **options)
recede_or_redirect_back_or_to(url, **options)
resume_or_redirect_back_or_to(url, **options)
refresh_or_redirect_back_or_to(url, **options)
```

The three native actions — `recede` (dismiss a modal or pop the stack), `resume` (ignore the navigation), `refresh` (reload the current screen) — are sent to native clients and fall back to an ordinary redirect for browsers. They are the server-side counterpart to bridge components: a way to affect native navigation without any native code.

Not provided by the framework, and defined in 3.3 of this guide: platform detection (`hotwire_native_ios?` / `hotwire_native_android?`) and app-version extraction.

User agent composition, appended automatically by the native libraries after your `applicationUserAgentPrefix`:

```text
<your prefix> Hotwire Native iOS; Turbo Native iOS; bridge-components: [a b c]; <WebView default UA>
<your prefix> Hotwire Native Android; Turbo Native Android; bridge-components: [a b c]; <WebView default UA>
```

## A.5 Naming quick reference

| Thing | Example | Must match |
|---|---|---|
| Stimulus identifier | `bridge--date-picker` | the `data-controller` attribute |
| `static component` | `date-picker` | iOS `name` and Android factory string |
| iOS `class var name` | `date-picker` | `static component` |
| Android factory string | `date-picker` | `static component` |
| Event name | `display` | the string passed to `reply(to:)` / `replyTo()` |

---

## A.6 The complete bridge stylesheet

Every CSS rule this guide adds, in one file. Two patterns only: hide what native replaces, reveal what only native can use.

```css
/* app/assets/stylesheets/bridge.css */

/* ---------------------------------------------------------------
   Elements that exist only for native apps. Hidden by default so a
   browser never sees them; revealed per component below.
   --------------------------------------------------------------- */
.bridge-native-only {
  display: none;
}

/* ---------------------------------------------------------------
   Hide the web control once the matching native component exists.
   The `~=` operator matches one item in the space-separated list
   the bridge writes onto <html>, so `~="form"` matches
   "form toast date-picker" but never "form-extras".
   --------------------------------------------------------------- */
[data-bridge-components~="form"] [data-bridge-hide-when-native],
[data-bridge-components~="toast"] [data-bridge-hide-when-native],
[data-bridge-components~="multi-select"] [data-bridge-hide-when-native],
[data-bridge-components~="typeahead"] [data-bridge-hide-when-native],
[data-bridge-components~="nested-form"] [data-bridge-hide-when-native],
[data-bridge-components~="wizard"] [data-bridge-hide-when-native] {
  display: none;
}

/* ---------------------------------------------------------------
   Reveal native-only affordances, one component at a time.
   Never write a single global rule for this: a page carrying a
   multi-select but no typeahead would reveal the wrong element.
   --------------------------------------------------------------- */
[data-bridge-components~="multi-select"] .bridge-native-only,
[data-bridge-components~="typeahead"] .bridge-native-only {
  display: block;
}
```

Components with **no** rule here — `confirm` (7), `select` (9), `date-picker` (10), `date-range` (12), `unsaved-changes` (17) — are the ones that replace nothing visible. The `<select>` and the date fields stay on screen in the app; native only changes what happens when you tap them. That absence is a good signal: a bridge component that needs no CSS is usually a well-scoped one.

If you move a field to a variant partial (3.5), delete its rules here. The variant decides what gets rendered, so there is nothing left to hide or reveal.

---

# Appendix B — Scope: device capabilities

This guide covers the complex *form inputs* — the components that replace custom JavaScript on data-entry controls, which is the bulk of what a browser-first Rails app needs. One cluster is out of scope, and it is a coherent one rather than a leftover: components that reach for **device capabilities**.

| Component | What it needs beyond this guide |
|---|---|
| **File & image upload** | A measured answer on whether current WebViews handle `<input type="file">` well enough — multi-select, direct camera capture — to make a bridge component unnecessary |
| **Camera: document scan & barcode** | A permission-rationale flow, and a decision on whether the captured file uploads from native or is handed to the WebView |
| **Signature capture** | PencilKit or a plain canvas, and whether the result goes through Active Storage or into the form as a blob |
| **Location & map picker** | Permission rationale, "while using" versus "always", and offline behavior |
| **Rich text / Action Text** | Confirmation that Trix's command surface drives reliably from a native accessory view |

They belong together because they share three questions, and answering those questions once serves all five.

### 1 · Does the upload go through JavaScript or through native HTTP?

The argument in 14.3 applies — the WebView owns the session, and duplicating authentication across two HTTP stacks is expensive. But files are large enough that a bridge round-trip may not be viable at all, which makes this the one place where native HTTP can be the right answer despite the auth cost. It needs measuring rather than reasoning.

### 2 · Who asks for permission, and when?

A permission prompt triggered by a bridge message has no visible cause from the user's point of view unless the web layer has already explained why it is coming. The rationale belongs on the web, where it can be written, translated and changed without a release; the prompt belongs to native.

### 3 · Where does a captured file live between capture and upload?

Native temporary storage, then either staged into the WebView or uploaded directly. The answer determines whether upload progress can be reported at all.

### One constraint that is already established

The Android WebView does not request runtime permissions on its own: `navigator.geolocation`, `getUserMedia`, and camera-backed file inputs fail **silently** in the app while working in Chrome on the same device (20.5, entry 12).

That removes an option. "Use the web API on Android, bridge only on iOS" would be the cheap route for location and camera, and it is not available — every component in this cluster requires native permission handling on Android before its web API works at all. This makes the cluster meaningfully more expensive than Chapters 11–17, and it is the first thing to confirm before designing any of it.

### Two components this guide names but does not build

Both come out of Chapter 15, and both are larger than the chapter that surfaced them.

- **Native keyboard accessory** — Previous / Next / Done walking focus across every field in a long form. On a twelve-row nested form this is a larger win than anything in Chapter 15. It is not covered here because `WKWebView`'s own input accessory view cannot be replaced through public API; the workable approach is a `UIToolbar` tracking keyboard-frame notifications, with an IME-inset equivalent on Android.
- **Row reordering** — drag-to-reorder is painful on mobile web and excellent natively, but it requires native row rendering, which puts row order into native state. Keeping Chapter 1's rule intact means the web form owns an explicit `position` field that native writes.

---

## Sources

- [Hotwire Native — Bridge Components reference](https://native.hotwired.dev/reference/bridge-components)
- [Hotwire Native — iOS Bridge Components](https://native.hotwired.dev/ios/bridge-components)
- [Hotwire Native — Android Bridge Components](https://native.hotwired.dev/android/bridge-components)
- [Hotwire Native — Bridge installation](https://native.hotwired.dev/reference/bridge-installation)
- [Hotwire Native — iOS configuration (user agent)](https://native.hotwired.dev/ios/configuration)
- [Hotwire Native — Android configuration (user agent)](https://native.hotwired.dev/android/configuration)
- [turbo-rails — `Turbo::Native::Navigation`](https://github.com/hotwired/turbo-rails/blob/main/app/controllers/turbo/native/navigation.rb)
- [hotwire-native-ios — `Demo/Bridge/FormComponent.swift`](https://github.com/hotwired/hotwire-native-ios/blob/main/Demo/Bridge/FormComponent.swift)
- [hotwire-native-ios — `Demo/Bridge/MenuComponent.swift`](https://github.com/hotwired/hotwire-native-ios/blob/main/Demo/Bridge/MenuComponent.swift)
- [hotwire-native-android — demo bridge components](https://github.com/hotwired/hotwire-native-android/tree/main/demo/src/main/kotlin/dev/hotwire/demo/bridge)
- [hotwire-native-bridge (web library)](https://github.com/hotwired/hotwire-native-bridge)
- [hotwire-native-ios releases](https://github.com/hotwired/hotwire-native-ios/releases)
- [Material Components — `MaterialDatePicker.Builder`](https://developer.android.com/reference/com/google/android/material/datepicker/MaterialDatePicker.Builder)

**Issue reports and diagnoses used in Chapter 20**

- [hotwire-native-ios#178 — snapshot cache shows stale content on redirect](https://github.com/hotwired/hotwire-native-ios/issues/178)
- [hotwire-native-ios#218 — page unresponsive on back navigation (tracked assets)](https://github.com/hotwired/hotwire-native-ios/issues/218)
- [hotwire-native-ios#239 — JS confirm/alert completion handler dropped](https://github.com/hotwired/hotwire-native-ios/issues/239)
- [hotwire-native-ios#244 — form submission redirects cancelled on the main stack](https://github.com/hotwired/hotwire-native-ios/issues/244)
- [hotwire-native-android#124 — missing browser events after navigation](https://github.com/hotwired/hotwire-native-android/issues/124)
- [hotwire-native-android#134 — WebView doesn't request camera/location permissions](https://github.com/hotwired/hotwire-native-android/issues/134)
- [hotwire-native-android#198 — missing flash messages](https://github.com/hotwired/hotwire-native-android/issues/198)
- [hotwire-native-bridge#10 — `disconnect()` not called on tab switch](https://github.com/hotwired/hotwire-native-bridge/issues/10)
- [Hotwire forum #6024 — bridge not working with Rails 7.2 (registration and import order)](https://discuss.hotwired.dev/t/unable-to-get-a-native-bridge-working-with-rails-7-2-and-hotwire-native-bridge-1-0-0/6024)
- [Jesse Waites — Debugging custom Bridge Components in Hotwire Native](https://jessewaites.com/blog/post/debugging-custom-bridge-components-in-hotwire-native/)
- [Joe Masilotti — bridge component library](https://masilotti.com/bridge-component-library/)
