# Manual render resolution: square fallback above 4096

The Companion saved 5376 × 4973 per eye correctly, and VRLink logged that exact
render target. SteamVR nevertheless received a 4096 × 4096 HMD recommendation;
150% area scaling then displays approximately 5016 × 5016. This is an independent
per-axis clamp, not a requirement that width be less than height.

Read-only analysis of the installed `driver_vrlink.dll`, SHA-256
`d415ba30fab129b4737ae19a3c71d7628c27e450dd8bf0002a1d4dd98ed58a05`,
identified the display recommendation routine at file offset `0x9e8a10`
(preferred VA `0x1809e9610`). It reads the two dimensions at `this+0x88` and
`this+0x8c`, clamps each to 100–4096, and writes them to the output pointers.
These offsets identify the inspected build only; no binary patch is used.
The running server's loaded GalaxyXRNative DLL matched the previously delivered
manual-resolution package hash, so this was not an older installed driver.
Connection and compositor logs provide evidence of the reported failure;
they do not establish that the corrected package has been loaded or tested.

The Galaxy XR shim now wraps the exact supported `IVRDisplayComponent` returned
by VRLink, identified by the module owning its vtable implementation. Other
drivers, missing components and other interface versions pass through. Its post
`GetRecommendedRenderTargetSize` callback restores the manual dimensions before
SteamVR applies resolution scaling. The original callback still runs, and all
other display methods retain their forwarding behavior.

The override is independent of native identity and the headset-profile switch.
Disabling it or deactivating the HMD preserves the original recommendation.
Explicit advanced VRLink resolution values/removals retain ownership of their
respective axes. Saved settings, defaults, refresh rate, projection, viewport,
stream encoding geometry and settings-journal ownership are unchanged. No
schema migration is needed.

`tools/Test-ManualRenderResolution.ps1` exercises the production parser, settings
writer, component wrapping, display dispatch macros and recommendation callback.
The fake upstream component reproduces the 4096 square fallback for 5376 × 4973;
regressions cover landscape, portrait, mixed above/below-limit axes, OFF/inactive
states, identity/profile independence, expert priority and unrelated components.

After installing the corrected package and restarting SteamVR, set Custom 100%
to compare the base dimensions. Auto, global and per-game scaling can change the
final pixel counts, and SteamVR can round dimensions. The driver logs one
recommendation correction per changed size for verification. A passing test or
package build is not a substitute for fresh headset/runtime verification.
