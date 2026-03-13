# Inventory Simulator Preview Notes

If your preview provider shows **Not Found** or **disallowed** for `Inv`, open:

- `index.html` (repo root)

Why: `Inv` is an extensionless legacy entry path kept for backward compatibility, while most preview systems only allow/render `.html` files directly.

## MILP note

The app attempts MILP using browser-loaded GLPK. In restricted preview/network environments where external scripts are blocked, it automatically falls back to DP and shows that in the solver status panel.
