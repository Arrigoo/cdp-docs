# Templating

Configuration values — webhook bodies, request URLs and headers, email
recipients, AI prompts and integration field mappings — are templates. Two
syntaxes are supported and both keep working:

- **Legacy placeholders**, e.g. `{{p.email}}`. Simple value substitution.
- **Templates**, e.g. `{{ .p.email }}`. Go `text/template`, which adds static
  values, combining several values, functions and conditionals.

The engine is chosen per value, automatically. Nothing needs migrating: an
existing configuration keeps rendering exactly as before until you rewrite it.

## Which engine renders a value

A value is rendered as a template if any of its `{{ … }}` actions starts with a
dot, a `$`, a `(`, a comment, or a function name. Otherwise it is rendered by the
legacy engine.

```
{{p.email}}         → legacy   (a bare word followed by a dot)
{{ .p.email }}      → template (leading dot)
{{ upper .p.name }} → template (function name)
{{ now }}           → template
```

**Do not mix the two syntaxes in one value.** The choice is made over the whole
string, so the losing syntax is passed through as literal text. The preview
dialog warns when a value mixes them, and saving reports it.

## What a template can read

| Root | Example | Notes |
|---|---|---|
| `.p` | `{{ .p.first_name }}` | Customer properties by `sys_title`. `{{ .p._cid }}` is the internal profile ID. |
| `.pv` | `{{ .pv.At "fav_loc" 0 }}`, `{{ .pv.Raw "fav_loc" }}` | List and object properties: one element, or the whole value as JSON. |
| `.c` | `{{ .c.Get "marketing" "email" }}` | Consent status. Defaults to `DENY` when nothing is recorded. `{{ .c.marketing_email }}` also works, and gives `""` when absent. |
| `.s` | `{{ .s.top_reader }}`, `{{ if .s.vip }}…{{ end }}` | Segment membership: the segment name when a member, empty otherwise. |
| `.i` | `{{ .i.email }}` | Profile identifiers. |
| `.x` | `{{ .x.api_key }}` | Secrets. Masked in previews. |
| `.ctx` | `{{ .ctx.Status }}`, `{{ .ctx.Get "body.recipient.email" }}` | The result of the action that triggered this follow-up. |
| `.res` | `{{ .res.Count }}`, `{{ .res.Get "paging.next" }}` | The previous API response, for paging URLs. |
| `.param` | `{{ .param.page }}` | Query parameters of the previous request URL. |
| `.src` | `{{ .src.Get "address.zip" }}` | One entry of a remote payload, in inbound field mappings. |
| `.inv` | `{{ .inv.Item "articles" "a-12" "title" }}`, `{{ .inv.Feed "top_reads" 0 "p.author" }}` | Catalog items and feeds. The trailing argument is a core field or `p.<sys_title>`. |

`.ctx`, `.res`, `.src` and `.pv` are read through methods (`Get`, `Raw`, `Val`,
`At`, `Has`) rather than dots, because their contents vary by payload. `Get`
returns text, `Raw` returns JSON, `Val` returns the value itself for piping into
a function, and `Has` is a presence test for `{{ if }}`.

**A missing value is never an error.** It renders as empty, exactly as the legacy
placeholders did, and `{{ if }}` treats it as false.

## Functions

`concat`, `default`, `join`, `split`, `upper`, `lower`, `title`, `trim`,
`trimPrefix`, `trimSuffix`, `replace`, `jsonstr`, `toJson`, `urlencode`, `now`,
`dateAdd`, `dateFmt`, `path`, `regexFind`, `int`, `add`, `sub`.

Go's own builtins are available too: `printf`, `index`, `len`, `slice`, `eq`,
`ne`, `lt`, `gt`, `and`, `or`, `not`, `urlquery`, plus `if`, `range`, `with` and
`$variables`.

### Use `jsonstr` in JSON bodies

Interpolating a value straight into a JSON body breaks the document as soon as
the value contains a quote or newline. `jsonstr` escapes it:

```json
{
  "name":  "{{ jsonstr .p.name }}",
  "email": "{{ jsonstr .p.email }}",
  "pages": {{ .p.pv_tot }}
}
```

The legacy engine has no equivalent, which is a long-standing source of failed
webhook deliveries.

It matters most for values you do not control — a field pulled from another
system's response, say:

```json
{ "res": "{{ jsonstr (.ctx.Get "body.someValue") }}" }
```

Without `jsonstr`, a quote anywhere in that value ends the JSON string early and
the remote rejects the request.

Note that the editor's JSON check understands template actions: the quotes around
a function argument, as in `{{ .ctx.Get "body.someValue" }}`, are template syntax
and are not counted as ending the surrounding JSON string.

### Use `urlencode` in URLs

```
https://api.example.com/search?q={{ urlencode .p.city }}
```

## Field mappings

Each mapping row is either a **single field** or a **template**, chosen with the
toggle next to the source field.

- **Single field** (the default) reads a dotted path from the payload and keeps
  the value's type — numbers stay numbers, lists stay lists. Typed and list
  destination properties depend on this.
- **Template** renders the source and always produces **text**.

Template mode is what makes these possible:

| Goal | Source |
|---|---|
| A static value | `imported` |
| A payload value plus a literal | `cdp:{{ .src.Get "id" }}` |
| Two payload values combined | `{{ .src.Get "first_name" }} {{ .src.Get "last_name" }}` |
| A fallback when a field is missing | `{{ .src.Get "nickname" \| default "friend" }}` |
| Part of a value | `{{ regexFind "[0-9]{4}" (.src.Get "address") }}` |

Outbound mappings (webhook and Agillic person data) work the same way, except
the template reads the customer profile rather than a remote payload:
`{{ .p.first_name }} {{ .p.last_name }}`.

Because template mode changes the value's type to text, it is opt-in per row
rather than inferred.

## Preview

The **Preview** button next to a body or template renders it on the server and
shows the result, optionally against a real profile ID or a sample payload.
Secret values are masked, so a preview can never disclose a credential.

Templates are also parsed when you save an integration or action; a syntax error
is rejected with a message naming the field, rather than failing later in the
action queue.

## Errors

- A template that does not parse, or that fails while rendering, **fails the
  action**. A partly-rendered payload is never sent.
- A missing value is not an error — it renders empty.
- In an inbound field mapping, a template that fails empties that one field and
  is logged, so one bad mapping cannot stop an entire import.

## Migrating a value

| Legacy | Template |
|---|---|
| `{{p.name}}` | `{{ .p.name }}` |
| `{{p._cid}}` | `{{ .p._cid }}` |
| `{{p.fav_loc.0}}` | `{{ .pv.At "fav_loc" 0 }}` |
| `{{p.fav_loc \| raw()}}` | `{{ .pv.Raw "fav_loc" }}` |
| `{{c.marketing}}` | `{{ .c.Get "marketing" "email" }}` |
| `{{s.top_reader}}` | `{{ .s.top_reader }}` |
| `{{i.email}}` | `{{ .i.email }}` |
| `{{x.api_key}}` | `{{ .x.api_key }}` |
| `{{ctx.status}}` | `{{ .ctx.Status }}` |
| `{{ctx.body.a.b}}` | `{{ .ctx.Get "body.a.b" }}` |
| `{{ii.articles.a1.title}}` | `{{ .inv.Item "articles" "a1" "title" }}` |
| `{{iif.top_reads.0.p.author}}` | `{{ .inv.Feed "top_reads" 0 "p.author" }}` |
| `{{req.time(+3d)}}` | `{{ dateAdd "+3d" }}` |
| `{{res.count}}` | `{{ .res.Count }}` |
| `{{res.a.b}}` | `{{ .res.Get "a.b" }}` |
| `{{param.page(+1)}}` | `{{ add 1 (int .param.page) }}` |

Rewriting is optional. Two things are worth rewriting for, beyond the added
expressiveness:

- **JSON safety.** The legacy engine pastes values in unescaped, so a quote in a
  property breaks the body. `jsonstr` fixes it.
- **Values are not re-scanned.** The legacy engine ran several substitution
  passes over its own output, so a profile property whose value happened to
  contain placeholder text was expanded by a later pass. Templates never re-scan
  what they produced.

## Note on consent placeholders

The legacy `{{c.*}}` placeholder never worked: its pattern rejected the second
dot, so `{{c.marketing.email}}` was left as literal text, and the shorter
`{{c.marketing}}` was compared against a key that always contained a dot, so it
always returned `DENY`. Use `{{ .c.Get "scope" "type" }}`, which resolves
properly and still defaults to `DENY`.
