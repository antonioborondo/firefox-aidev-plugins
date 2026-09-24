# Stub installer GUI and WebBrowser plugin

## GUI constraints
- IE8 rendering mode (must work on unpatched Windows 7)
- Image assets: JPEG, GIF, PNG only (no WebP, no SVG)
- Fonts limited to default Windows 7+ fonts
- Window size is fixed; changing it affects all brandings
- Must maintain accessibility (ARIA attributes for custom controls like the
  progress bar)

## WebBrowser plugin API
- `ShowPage` — display a URL/file in the embedded browser; blocks until
  page closes
- `RegisterCustomFunction` — expose an NSIS function to JavaScript via
  `window.external.functionName()`; takes function address + name; call
  before `ShowPage`
- `CreateTimer` — create an interval timer with callback; returns handle
- `CancelTimer` — stop a timer by handle

Custom functions take one string parameter from the NSIS stack and return
one string. JavaScript receives/returns strings; conversions happen in JS.
