# Live endpoint checker (check_mcp.py)

Write the code block below to `check_mcp.py` (Python 3, stdlib only) and run `python check_mcp.py https://your.site/mcp --call TOOL '{"arg":"x"}' --expect-error TOOL '{}'`.

```python
"""Live check of a ChatGPT plugin MCP endpoint (stdlib only).

python check_mcp.py https://example.com/mcp [--call TOOL 'JSON_ARGS'] [--expect-error TOOL 'JSON_ARGS']
Exit 0 = all checks passed.
"""
import json, sys, urllib.request

def rpc(url, method, params, i):
    body = json.dumps({"jsonrpc": "2.0", "id": i, "method": method, "params": params}).encode()
    req = urllib.request.Request(url, body, {"Content-Type": "application/json",
                                             "Accept": "application/json, text/event-stream"})
    with urllib.request.urlopen(req, timeout=30) as r:
        raw = r.read().decode()
    if raw.lstrip().startswith("event:") or "data:" in raw[:20]:
        raw = next(l[5:].strip() for l in raw.splitlines() if l.startswith("data:"))
    return json.loads(raw)

def main(a):
    url, fails = a[0], []
    init = rpc(url, "initialize", {"protocolVersion": "2025-06-18", "capabilities": {},
                                   "clientInfo": {"name": "check_mcp", "version": "1"}}, 1)
    print("initialize:", json.dumps(init.get("result", init))[:300])
    if "error" in init: fails.append("initialize")
    tools = rpc(url, "tools/list", {}, 2).get("result", {}).get("tools", [])
    for t in tools:
        d = t.get("description", "")
        print(f"tool {t['name']}: {d[:100]}")
        if not d.lower().startswith("use this when"): fails.append(f"{t['name']}: description must start 'Use this when'")
        if t["name"].lower() in ("best", "official", "pick_me"): fails.append(f"{t['name']}: banned name")
    if not tools: fails.append("no tools")
    k = 3
    while "--call" in a or "--expect-error" in a:
        flag = "--call" if "--call" in a else "--expect-error"
        j = a.index(flag); name, args = a[j + 1], json.loads(a[j + 2]); del a[j:j + 3]
        r = rpc(url, "tools/call", {"name": name, "arguments": args}, k); k += 1
        out = json.dumps(r.get("result", r), ensure_ascii=False)
        print(f"{flag} {name}: {out[:600]}")
        is_err = "error" in r or r.get("result", {}).get("isError")
        if flag == "--call" and (is_err or not r.get("result", {}).get("content")): fails.append(f"call {name} failed/empty")
        if flag == "--expect-error" and not is_err: fails.append(f"{name} accepted bad input")
    print("FAIL:\n  " + "\n  ".join(fails) if fails else "PASS")
    return 1 if fails else 0

if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```
