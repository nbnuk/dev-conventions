# HTMX Patterns

## Principle

Keep interactions server-driven. Use HTMX for partial updates and events, not for rebuilding SPA-style client state.

This applies to every screen and every components-plugin component, and it is the default for a first draft, not a later clean-up:

- **The server renders the data.** Lists, choices, labels and messages come back as HTML from an endpoint. Don't send JSON for the page to build markup from, and don't keep a client-side copy of server data that then has to be kept in sync.
- **Requests are HTMX attributes.** Use `hx-get` / `hx-post` with `hx-vals` / `hx-include` on the element. Don't use `fetch` and don't script requests.
- **The server updates the rest of the page.** Use out-of-band swaps (`hx-swap-oob`, which can target every element matching a selector) or `HX-Trigger` events that make other fragments refetch. Don't use custom events that carry data for a script to apply.
- **Script only for what HTMX can't do:** positioning a popup, moving focus, keyboard handling (arrow keys, Escape) and closing on a click outside. Keep it small, delegated on `document` so swapped-in content works, and free of data.
- **Wait for settle before using swapped content.** HTMX activates new content at settle, not at swap. A script that shows or focuses swapped-in controls listens for `htmx:afterSettle`. On `htmx:afterSwap`, a quick Enter can submit a form as a normal page load.

Example: the components plugin's single term picker. Its row toggles GET their choices into a shared popup, a pick POSTs and replaces the row, and an out-of-band button redraws every row. Its script only shows, places and closes the popup.

## Core patterns

### 1) Form fragment + server validation rerender

Render a form fragment into a container, submit via HTMX, and rerender the same fragment on validation errors.

```html
<div id="userFormContainer"></div>
<button hx-get="/admin/user/formPartial" hx-target="#userFormContainer" hx-swap="innerHTML">
  Create user
</button>
```

```groovy
if (cmd.hasErrors()) {
    render template: 'form', model: [cmd: cmd]
    return
}
```

### 2) Success via `HX-Trigger` (not inline JS)

On successful HTMX POST, return `HX-Trigger` JSON and an empty body.

```groovy
response.setHeader('HX-Trigger', JsonOutput.toJson([
    userSaved: [id: user.id.toString(), name: user.name, message: 'User saved.']
]))
render ''
```

Use `JsonOutput.toJson(...)`; do not hand-build JSON strings.

### 3) Refresh list partial from event

Keep list/table in a dedicated container that refetches on a domain event.

```html
<div id="userListContainer"
     hx-get="/admin/user/listPartial"
     hx-trigger="userSaved from:body"
     hx-swap="innerHTML">
  <g:render template="userList" model="[users: users]"/>
</div>
```

`from:body` is important because `HX-Trigger` events bubble and are commonly observed from `body`.

### Endpoint naming convention

Use controller action names to describe response shape without encoding transport concerns in URL prefixes:

- Fragment-only actions use a `Partial` suffix (`formPartial`, `listPartial`).
- Full-page actions use unsuffixed domain names (`index`, `show`, `edit`).
- Reserve neutral unsuffixed names for endpoints that intentionally serve both full and partial responses based on request context.

### 4) Reuse same event in multiple contexts

Use one event name (for example `userSaved`) for both:
- standalone admin page list refresh
- modal/wizard listeners that update local UI

```js
document.body.addEventListener('userSaved', function (evt) {
  const d = evt.detail;
  if (!d || !d.id) return;
  // e.g. append new option, close modal, etc.
});
```

### 5) Success feedback: toasts for confirmations, alerts for what the user must keep

Choose by what the message is for, not by how the request was made. Users can't
see whether a page used HTMX, so the same kind of message must look the same.

- **Confirmations** ("Person created", "Changes saved", "3 people deleted",
  "Import discarded") are always a **success toast**.
- **Messages the user must read, keep or act on** are a **page alert**, because
  a toast disappears after a few seconds. Examples: a temporary password, "This
  reset link is invalid or has expired", "Choose the file encoding and separator
  before continuing".

`<components:toast/>` (nbn-components-plugin) shows a confirmation from either
route, with no page code:

- **HTMX response:** put `message` on the domain event.

  ```groovy
  response.setHeader('HX-Trigger', JsonOutput.toJson([userSaved: [message: 'User saved.']]))
  render ''
  ```

- **Full-page post that redirects (PRG):** set `flash.toast`; the next page shows it.

  ```groovy
  flash.toast = "Person “${person.displayName}” created."
  redirect action: 'people'
  ```

Use `flash.message` (success style) and `flash.error` only for page alerts.

For server failures on HTMX requests, handle `htmx:responseError` once in layout.

```js
document.body.addEventListener('htmx:responseError', function (event) {
  const xhr = event.detail.xhr;
  let text = xhr.status + ' ' + (xhr.statusText || '');
  const ct = (xhr.getResponseHeader('Content-Type') || '').toLowerCase();
  if (ct.includes('application/json') && xhr.responseText) {
    try {
      const body = JSON.parse(xhr.responseText);
      text = body.error ? (xhr.status + ': ' + body.error) : text;
    } catch (e) { /* keep fallback text */ }
  }
  if (window.showErrorToast) window.showErrorToast(text, 'Server error');
});
```

## Controller response rule of thumb

- HTMX success: `HX-Trigger` (with `message` for a confirmation) + `render ''`
- HTMX validation error: rerender fragment with errors
- Non-HTMX success: `flash.toast` + redirect (PRG); `flash.message` only when the user must keep or act on the message

## What to avoid

- Returning `HX-Redirect` for every HTMX success when a partial refresh is enough
- Mixing many one-off event names for the same domain action
- Building `HX-Trigger` JSON by string concatenation
- Using flash messages as the primary HTMX feedback channel
- A page alert for a plain confirmation, or a toast for something the user must keep or act on
