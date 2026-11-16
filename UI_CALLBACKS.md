# UI callbacks and custom JavaScript forms — checkout compatibility

This Deployment Documentation note records behavior verified in the pinned Open WebUI checkout identifying itself as 0.13.0. Use the **Official Open WebUI Documentation** KB for the full Tool, Filter, Action and Pipe interfaces. This note supplements those interfaces; it does not change the official mirror or claim compatibility with every release.

## Choosing the execution owner

Use a **workspace Tool** when the model should choose when to open a form; an **Action Function** when the user should click a message button; a **Filter Function** when a lifecycle condition should trigger processing or an interaction; a **Pipe Function** when the implementation owns response generation/orchestration. Attach tools, actions and filters through Model Management's corresponding selection fields. A Pipe is selected by its exposed model ID as `base_model_id`, not an attachment. None of these choices requires inventing a new frontend event type.

For a named existing custom tool, read its own page first. Ask User (deployment checkout: `Michael/docs/tools/user_clarification.md`) already provides choice/text questions. Review and send email (deployment checkout: `Michael/docs/functions/actions/mail_review.md`) demonstrates a custom multi-field form. Creating a new tool is unnecessary when an existing one meets the request.

## Emitter versus caller

Accept the injected `__event_emitter__` and/or `__event_call__` parameters in the supported method signature. Open WebUI supplies them from the authenticated chat context; do not invent session IDs, borrow another user's token or construct a client-supplied identity.

| Mechanism | Purpose | Examples supported by this checkout |
| --- | --- | --- |
| `await __event_emitter__(event)` | Send a one-way update; it is not an answered question | `status`, `notification`, message/file/source updates using the corresponding frontend payload contracts |
| `await __event_call__(event)` | Send an interaction and await its returned value | `confirmation`, `input`, `request:user_input`, `request:elicitation`, `execute` |

Event names and payload shapes are defined by the frontend handlers. An invented event name does not create a new UI. An Action Function is a resource type, not a prerequisite for using callbacks from another supported extension. Callbacks require a connected, authenticated session for the requesting user; headless requests may have no caller or return a disconnected-session error. Treat absence, dismissal, timeout and invalid results as incomplete input, not approval.

## Custom forms through `execute`

The supported `execute` event accepts `data.code`. The frontend evaluates that JavaScript inside an async function and returns its resolved value through the socket callback. Code can create DOM elements, open a form and `return` a Promise that resolves on submit or cancel. There is no need to register a new frontend Action type for the form itself.

Minimal extension-side pattern (supply reviewed JavaScript, not user-provided executable text):

```python
result = await __event_call__({
    "type": "execute",
    "data": {"code": reviewed_form_javascript},
})
```

Custom form implementation contract:

1. Create elements explicitly and populate user values with `textContent`/`value`; serialize data safely rather than interpolate it into JavaScript or HTML.
2. Provide labels, keyboard focus, required-field validation, a Submit button and a Cancel/close path. Resolve a structured result such as `{status: "submitted", values: {...}}` or `{status: "cancelled"}`.
3. Resolve once, remove created DOM and listeners, and clear timers on every exit. Bound browser lifetime and server wait; the first timeout/disconnection may end the interaction. Catch failures and return an explicit error/cancelled state rather than leaving an unresolved Promise.
4. Validate returned values again on the backend before any mutation. Browser validation and UI text do not grant authorization. Preserve caller identity, resource ownership and exact-operation confirmation requirements.
5. Use the returned object as the tool/action result or deliberately handle it in the owning filter/pipe. Opening a modal does not itself insert a tool call, tool result or new user turn into conversation history.
6. Test submit, cancellation, invalid input, repeated opening and cleanup. Test the actual browser integration separately from source installation and API read-back.

`execute` runs JavaScript in the application page; it is not a sandbox. Avoid unrelated DOM changes, external network requests, storage/token access and new dependencies. Standard input/confirmation callbacks are simpler when they fit the requested interface. The frontend has an `execute` handler, but individual clients or browser policies may differ; a delivered event alone does not prove successful rendering.

## Filters, streaming and further model calls

In this checkout, inlet/request/outlet hooks receive the injected context supported by their signatures; inspect the exact hook before relying on a parameter. Inlet rejection normally surfaces as an error, not a normal canned assistant reply. An outlet can rewrite a finished reply, but streamed text may already have appeared. Preserve structured message/output arrays when rewriting: `output` message `content` is an array of text parts, not a string.

Filter Python code can explicitly invoke an available backend helper/model request or UI callback. That is different from **returning an edited body**, which does not restart the native model/tool loop. Additional requests need recursion guards, bounded retries, explicit response handling and preserved caller permissions. A Pipe is generally the clearer owner when the task requires deterministic no-model responses or control of repeated model/tool turns. Do not describe a particular generated filter's limitations as universal platform restrictions.

## Deployment checkout verification references

- `src/lib/components/chat/Chat.svelte`: frontend event handling for `execute`, confirmation, input, notifications and structured input interactions.
- `backend/open_webui/socket/main.py:get_event_call`: authenticated session ownership, callback result transport and timeouts.
- `backend/open_webui/utils/middleware.py:outlet_filter_handler`: injected caller/emitter and finished-response processing.
- `backend/open_webui/utils/filter.py`: method-signature injection and lifecycle dispatch.
- `Michael/tools/ask_user.py` and `Michael/functions/mail_review.py`: custom forms already used by this deployment.

Manage source with Tool Management (deployment checkout: `Michael/docs/tools/admin_tools.md`), Filter Management (deployment checkout: `Michael/docs/tools/admin_filters.md`), Action Management (deployment checkout: `Michael/docs/tools/admin_actions.md`) or Pipe Management (deployment checkout: `Michael/docs/tools/admin_pipes.md`); configure preset bindings with Model Management (deployment checkout: `Michael/docs/tools/admin_models.md`).
