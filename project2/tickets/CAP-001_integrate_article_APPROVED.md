# CAP-001: Integrate "The Cost of Waiting"

**Status suffix:** `_APPROVED`

*(SAMPLE TICKET - content lifecycle. Note the status is a filename **suffix**, not a bracket prefix. Bracket prefixes are reserved for article content states, so the two never collide.)*

```
_APPROVED -> _VERIFIED -> _RFT -> _FIXED
                 ^          |
                 +--- _FIX_FAILS (Tester found failures; back to Implementer)
Blocked: _CANNOT_TEST (Tester) / _CANNOT_IMPL (Implementer) -> Designer reviews
```

## Notes to Tester

Write failing assertions for: the article renders at its slug; it appears in the library listing; the prev/next chain still runs unbroken; the total count went up by exactly one. Then rename this file `_VERIFIED`.

## Notes to Implementer

Edit `src/` and the build script only. **Never hand-edit `dist/`.** Never modify `rtest.py` - that belongs to the Tester. When the build assertions and rtest pass, rename to `_RFT`.

## Definition of Done

- [ ] Article reachable at its slug on phone, tablet and desktop, light and dark
- [ ] Standing assertions green
- [ ] Committed and pushed