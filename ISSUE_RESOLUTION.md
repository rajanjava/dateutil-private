# Issue Resolution: Raise ValueError instead of IndexError in rrulestr without RRULE

## 1. Upstream Issue Reference
- **Repository:** `dateutil/dateutil`
- **Issue:** [#1668](https://github.com/dateutil/dateutil/issues/1668) - `rrulestr raises IndexError for a string with DTSTART but no RRULE`
- **Status:** Open upstream

## 2. Problem Description
When parsing an RFC recurrence string that contains a `DTSTART` line (or other header properties) but no recurrence rule (`RRULE`, `RDATE`, `EXRULE`, or `EXDATE`):
```python
from dateutil.rrule import rrulestr

rrulestr("DTSTART:20200101")
```
It crashed with an unhandled:
`IndexError: list index out of range`
instead of raising a `ValueError`, which is standard for malformed recurrence strings in `rrulestr` (e.g. `rrulestr("")`).

### Root Cause
In `src/dateutil/rrule.py`, `_rrulestr._parse_rfc()` checked:
```python
if (forceset or len(rrulevals) > 1 or rdatevals or exrulevals or exdatevals):
    ...
else:
    return self._parse_rfc_rrule(rrulevals[0], ...)
```
When `forceset` was `False` and no rules were provided, `rrulevals` was empty (`len(rrulevals) == 0`). The `else` branch unconditionally accessed `rrulevals[0]`, raising an unhandled `IndexError`.

## 3. Reproduction Test Case
Added in `tests/test_rrule.py`:
```python
    def testStrWithoutRRULE(self):
        with self.assertRaises(ValueError):
            rrulestr("DTSTART:20200101")
        with self.assertRaises(ValueError):
            rrulestr("DTSTART:20200101\n")
```

Prior to the fix:
`IndexError: list index out of range`

## 4. Solution & Code Changes
In `src/dateutil/rrule.py`:
- Updated the `else` branch to `elif rrulevals:` before parsing `rrulevals[0]`.
- Added an `else:` branch raising `ValueError("No RRULE found")` when no recurrence rule is present.

## 5. Verification
- Reproduction test passed.
- All 739 test cases in `tests/test_rrule.py` passed (`.venv/bin/pytest tests/test_rrule.py`).
