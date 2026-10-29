# Widget registry and examples resource

Applications with registered `@widget` tools expose a JSON catalog at
`widget://examples`. Clients discover it with `resources/list` and read it with
`resources/read`. Existing `ui://widget/...html` resources continue to serve HTML.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "resources/read",
  "params": {"uri": "widget://examples"}
}
```

Use normal client initialization or modern protocol metadata and HTTP headers,
as for any resource read. The response text has MIME type `application/json`:

```json
{
  "widgets": [
    {
      "route": "calculator-result",
      "uri": "ui://widget/calculator-result.html",
      "name": "calculate",
      "description": "Calculate a result",
      "mime_type": "text/html;profile=mcp-app",
      "metadata": {},
      "tool_names": ["calculate"],
      "examples": [
        {
          "tool_name": "calculate",
          "input": {"a": 2, "b": 3},
          "output": {"result": 5},
          "description": "Add two numbers"
        }
      ]
    }
  ],
  "count": 1,
  "loaded": true
}
```

Widget MIME types follow `NITROSTACK_APP_MODE`: `text/html` for OpenAI mode and
`text/html;profile=mcp-app` for MCP App/universal mode. Metadata uses the existing
mode-aware resource builder, including declared CSP, domain and border settings.
Configure the mode before creating the app; the registry is a startup snapshot.

## Declaring examples

Existing `ToolExamples` declarations supply both input and output. The registry
never calls tools to generate data or scans source files.

```python
from pydantic import BaseModel
from nitrostack import ExecutionContext, ToolExamples, tool, widget

class AddInput(BaseModel):
    a: float
    b: float

class CalculatorTools:
    @tool(
        "calculate", "Calculate a result", AddInput,
        examples=ToolExamples(
            input={"a": 2, "b": 3},
            output={"result": 5},
            description="Add two numbers",
        ),
    )
    @widget("calculator-result")
    async def calculate(self, input: AddInput, context: ExecutionContext):
        return {"result": input.a + input.b}
```

Register the provider/controller in a module as usual. String decorators, option
mappings, `WidgetOptions` and helpers returning options are all supported. Tools
without examples have an empty examples list. Explicit `None`, `False`, zero and
empty collections are retained. Use JSON-compatible values. Examples are public
catalog data: do not declare real credentials or private user data in them.
Widget HTML and injected environment credentials are not included.

## Python access

```python
from nitrostack import get_widget_registry

registry = get_widget_registry(app)  # identical to app.widget_registry
widgets = registry.get_all_widgets()  # list[WidgetMetadata]
widget = registry.get_widget("ui://widget/calculator-result.html")
manifest = registry.get_manifest()  # WidgetManifest
data = manifest.to_dict()
```

`WidgetExample`, `WidgetMetadata`, `WidgetManifest`, `WidgetRegistry` and
`get_widget_registry` are public in both `nitrostack` and `nitrostack.widgets`.
Missing URI lookups return `None`. Records are frozen dataclasses; nested lists
and dictionaries returned by reads are detached copies. Changing them cannot
alter later reads. Each application owns an independent registry.

## Shared routes and empty applications

Each normalized route appears once, sorted lexicographically. Tool names and
examples are sorted by tool name, retaining every declared example. The first
tool name supplies the display name, description and component metadata. This
selection does not change the existing HTML resource's rendering behavior.

`count` is the unique widget count. `loaded` means startup discovery completed,
including for an empty registry. Repeated reads are stable; no timestamps are
generated. Applications without widgets have an empty Python registry and no
additional MCP resource. A user-defined `widget://examples` is left alone in
such apps. Widget applications reserve that URI and reject collisions at startup.

The manifest describes registered application widgets. It introduces no session
filtering or authorization policy and grants no tool execution permissions. The
application's existing resource access controls still apply.

## Implementation explanation

The registry module separates data descriptions from HTML rendering:

- The `TYPE_CHECKING` import supplies the application annotation without a
  runtime circular import. `WIDGET_EXAMPLES_URI` names the built-in resource.
- `WidgetExample` retains tool name, input, output and optional description.
  It never tests example payloads for truthiness.
- `WidgetMetadata` holds one route's URI, MIME type, metadata and tool examples.
  Default factories give each record independent mutable fields.
- `WidgetManifest` contains widgets, count and completion status. `to_dict()`
  uses `asdict()` to recursively copy the dataclass records into dictionaries.
- `WidgetRegistry.__init__()` sorts records and deep-copies them into a private
  URI-keyed mapping. Later changes to source declarations cannot affect it.
- `_from_tools()` visits final registered tool entries, including only entries
  with components. Python has already evaluated all decorator helpers. Grouping
  by component ID reuses existing route normalization. Sorting groups and tool
  names makes output independent of registration order.
- The example comprehension copies only explicit tool examples. Each group's
  first component supplies name, description and URI. Existing helpers supply
  mode-correct MIME types and resource metadata. No HTML enters the catalog.
- `get_all_widgets()` and `get_widget()` deep-copy the private records.
  `get_manifest()` builds on those detached reads and computes their count.
  Registry construction is complete at that point, so `loaded` is true.
- `get_widget_registry(app)` returns the application attribute rather than a
  global singleton, preventing cross-application catalog leakage.

Application bootstrap constructs the registry after registering decorated
methods and before installing MCP handlers. A nonempty registry adds one ordinary
resource entry. Its handler returns the manifest dictionary; the existing
resource pipeline serializes it as JSON. Additional configured MCP servers use
the same resource registrations. Existing widget HTML handlers are unchanged.
Package exports expose the types and accessor through both public entry points.

Integration tests bootstrap actual templates in subprocesses, blocking socket
networking so startup cannot depend on external providers. HTTP tests exercise
modern/sessionless and legacy/sessionful modes. A separate stdio process answers
real resource requests with JSON-RPC only on stdout. Test tools raise if invoked,
establishing that catalog operations never execute them.
