# PHP Patch Loop Bug: `sa` type not applying `q`/`ans`

## The Bug (discovered 2026-07-21, mã 1182 r2)

In `ttp_patch_1182_r2_v1()`, the loop only applied `['e']` (barem) for ALL sa questions:

```php
// BUG: chỉ apply e, không apply q/ans cho sa4
if ( 'sa' === $type && isset( $sa[ $no ] ) && false === mb_strpos( $e, $sa[ $no ]['guard'] ) ) {
    $data[ $i ]['e'] = $sa[ $no ]['e'];  // only e!
    $changed         = true;
}
```

**Result**: For sa4 (which changed both q+ans+e), only the barem was written. The question stayed as "7x = 0" → Gemini still saw the old question → TC10 kept failing.

## The Fix

```php
$q    = (string) ( $it['q'] ?? '' );
$e    = (string) ( $it['e'] ?? '' );

// sa4 đổi cả q/ans/e ⇒ guard theo q; sa1/sa2/sa5 chỉ thêm barem ⇒ guard theo e
$haystack = isset( $sa[ $no ]['q'] ) ? $q : $e;
if ( false !== mb_strpos( $haystack, $sa[ $no ]['guard'] ) ) {
    continue;
}
if ( isset( $sa[ $no ]['q'] ) ) {
    $data[ $i ]['q'] = $sa[ $no ]['q'];
}
if ( isset( $sa[ $no ]['ans'] ) ) {
    $data[ $i ]['ans'] = $sa[ $no ]['ans'];
}
$data[ $i ]['e'] = $sa[ $no ]['e'];
$changed         = true;
```

## Rule: 3 types of sa patching

| Type | What changes | Guard on | Apply fields |
|---|---|---|---|
| **barem-only** (sa1/sa2/sa5) | `e` only | `e` (barem text) | `e` only |
| **full replace** (sa4) | `q` + `ans` + `e` | `q` (question text) | `q`, `ans`, `e` |
| **q-only** (rare) | `q` only | `q` | `q` (preserve ans/e) |

## Bumping gate options (deploy fix)

When fixing a bug in an already-deployed patch whose gate option was already set on the server:

```php
// OLD (already set on server):
if ( get_option( 'ttp_patch_1182_r2_v1' ) ) { return; }

// NEW (bump name so server runs again):
if ( get_option( 'ttp_patch_1182_r2b_v1' ) ) { return; }
```

Must also bump the `update_option()` call to match.

## Guard selection rule

- `guard` must be a substring **unique to the new content**, not present in the old content.
- For **barem-only**: `guard` in `e` (new barem text like "Bước 4 (0,25đ):" or "AH² = BH·CH")
- For **full-replace**: `guard` in `q` (part of new question text like "(2x − 3)/2")
- **Never** use the same guard for both types in the same question.
