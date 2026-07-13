AERO Codebase & Architecture Review

Prepared by: Senior Software Architect & Rust Engineer Status: Read-Only Analysis (Phase 1b Complete)
1. Executive Summary

Aero is an exceptionally well-engineered, highly secure, and clean-slate dual-engine web browser written in Rust. 
It decouples core browser concerns (filtering, session management, identity, and navigation state) from the rendering process. 
It is currently mid-flight in its roadmap (Phase 1b complete, preparing for Phase 2).
Architectural Health Score: 9.5 / 10

The Good: Outstanding separation of concerns. 
The Engine trait boundary is hermetically sealed; neither the shell nor the filtering logic leaks engine-specific details. 
Conformance testing acts as an objective, data-driven referee between engines. 
High-quality documentation (CEF-GAPS.md, SERVO-GAPS.md, ARCHITECTURE.md) shows rare self-awareness of upstream dependencies 
and system design tradeoffs.
    The Gaps: Deep integration gaps exist in the upstream servo-net and embedding API layers 
(e.g., lack of isolated contexts, proxy settings bypass, and a severe subresource integrity panic) which prevent Servo from acting 
as a secure primary engine today. This justifies keeping CEF as the default engine until Milestone M4.

2. Key Architectural Innovations & Design Decisions
A. The Hermetic Engine Trait Boundary

The core abstraction in crates/engine-api/src/lib.rs isolates the browser shell from the rendering engines (CefEngine and ServoEngine).

    Offscreen Compositing: Neither engine is permitted to own native OS windows. 
They render windowless, exposing a row-major, top-left origin pixel Frame buffer. 
Input is routed back into the engine via explicit InputEvent coordinates. This enables natively parallel split-view and tab-hover 
previews without leaking browser chrome logic.
    Opaque Session Blobs: Context storage is managed using an engine-blind SessionBlob byte vector, allowing client-side encryption 
without knowing whether CEF or Servo produced the cookie jar.

B. Double-Fenced Network Filtering Layer

Every network request goes through a unified pipeline in crates/filter/src/lib.rs that applies:

    Request Interceptor Stage: A high-speed pipeline of stages (SecurityBlocklistStage first to prioritize overrideable interstitials, 
then AdblockStage powered by adblock-rust).
    In-Process Local DoH Proxy: Real engines cannot bypass DNS-over-HTTPS. 
Both engines are pointed at an in-process, loopback-only filtering proxy (filter::doh_proxy), backed by a custom hickory-resolver client. 
If a domain is blocked, DNS lookup is refused on loopback before ever leaking the plaintext query to an upstream server.

C. Conformance Testing as an Objective Referee

Rather than relying on manual validation, crates/conformance implements a data-driven scenario suite (Scenario registry) that runs 
identically against:

    MockEngine (fast, in-memory, used as a synchronous CI gate).
    ServoEngine and CefEngine (real engine instances, async, running on real network pipelines).

Pass/partial/fail metrics determine milestone readiness, defining a truly objective path toward deleting CEF (Milestone M4).
3. Codebase Quality & Idiomatic Rust Analysis

    Clean and Highly Modular Layout: The workspace is perfectly divided into single-purpose crates (engine-api, engine-servo, engine-cef, 
filter, conformance, and the aero-browser shell).
    Deterministic Startup & Zero Allocation on Hot Paths:
        Filter stages load rules from local disk caches synchronously before updaters spawn, ensuring Tab 0 is never loaded unfiltered.
        Evaluator reasons use Cow<'static, str> instead of plain String to prevent costly heap allocations on clean requests 
(100 µs hot-path budget).
    Robust Debouncing & Throttling: Rapid window resizes are throttled to a max rate of 10Hz (RESIZE_THROTTLE of 100ms) 
inside AeroApp::render_active_tab to prevent libcef.dll from crashing on rapid layout adjustments.
    No-Leak Event Model: The ChannelListener pattern decouples callbacks (firing deep inside engine threads during pump) 
from the application event loop by routing them through standard mpsc channels.

4. Critical Technical Gaps & Landmines

Through our analysis of the codebase, SERVO-GAPS.md, and CEF-GAPS.md, we highlight the following critical issues that the team must track closely:
A. The Servo Subresource Integrity (SRI) Thread Panic

    The Issue: A page containing CDN-hosted third-party scripts with integrity attributes (e.g., canyoublockit.com/extreme-test/) 
panics a Servo worker thread if a filter stage returns a Verdict::Block or Verdict::Redirect.
    The Cause: Upstream servo-net-0.3.0's apply_algorithm_to_response asserts that the response body is already fully received, 
yet the embedder API WebResourceResponse has no body field at all, making it impossible to pass a completed body.
    Impact: High crash risk. This alone prevents Servo from being safe for arbitrary real-world web browsing today.

B. Servo Proxy Settings Bypass (Unresolved)

    The Issue: While ProxyConnector in servo-net claims to read global proxy settings, pointing real ServoEngine navigations 
at the local loopback proxy does not route traffic through it. Real pages bypass the proxy and connect directly to the internet.
    Impact: Servo tabs do not currently benefit from the local DoH resolver or resolution-time blocking.

C. CEF Single-Process Initialization Limitation

    The Issue: CEF allows exactly one initialize and shutdown cycle per OS process. Instantiating a second CefEngine triggers an atomic panic.
    Impact: Conformance tests for CEF must run with --test-threads=1 and be isolated. Additionally, the shell must share a single 
CefEngine singleton across all tabs.

5. Transition to Phase 2 (Local Data Platform) & Recommendations

As the project transitions to Phase 2, the currently skeletonized crates (vault, storage, identity, bridge-1p) will be populated. 
Here is an architectural blueprint for their implementation:

    Session Vault (crates/vault):
        Use client-side XChaCha20-Poly1305 symmetric encryption to wrap cookie/storage snapshots (SessionBlob) before they hit the database.
        Derivation of the Data Encryption Key (DEK) must use HKDF backed by X25519 ECDH key agreement between the paired phone and the 
local device profile.
    Storage Layer (crates/storage):
        Keep it isolated! Utilize sqlx with mandatory TLS.
        Implement the "N3 / Ephemeral Mode" graceful degradation: if PostgreSQL is unreachable, automatically swap out the real database 
handle for an in-memory fallback.
    Credentials Bridge (crates/bridge-1p):
        Integrate securely with the op CLI.
        Enforce a strict Anti-Phishing Address Matching rule: Only allow credential requests where the hostname perfectly matches 
the active tab's browser address bar.
        Use the zeroize crate to explicitly overwrite memory buffers containing autofilled credentials immediately after execution.

Conclusion

Aero is a masterclass in browser architecture. The engineering practices (conformance reports, gap documentation, 
and absolute engine isolation) set a gold standard for Rust development. Mitigating the upstream Servo proxy and SRI bugs are the 
highest priority items before Servo can graduate from an opt-in compatibility engine to the default.

Let me know if you would like me to dive deeper into any of these modules!
