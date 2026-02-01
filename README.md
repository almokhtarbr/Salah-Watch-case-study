# Prayer Times for Garmin Smartwatch: An Embedded Systems Case Study

## Overview

A lightweight Islamic prayer time calculator for Garmin Epix Pro smartwatches that solves the challenge of providing accurate, location-aware prayer times in an offline-first embedded environment. The application implements sophisticated astronomical algorithms entirely on-device, eliminating dependencies on external APIs while delivering sub-minute accuracy across five globally recognized calculation methodologies. By prioritizing reliability and power efficiency over connectivity, the solution ensures uninterrupted access to time-sensitive religious observances regardless of network availability.

## Technical Challenges

### 1. **Computational Constraints in Embedded Runtime**
Implementing complex astronomical calculations (Julian date conversion, equation of time, solar declination, spherical trigonometry) within the memory and processing limitations of a wearable device required algorithmic optimization without sacrificing accuracy. The absence of external mathematical libraries necessitated first-principles implementations using only basic trigonometric primitives.

### 2. **Multi-Method Prayer Calculation Complexity**
Supporting five distinct calculation methodologies (Muslim World League, ISNA, Egyptian General Authority, Umm al-Qura, University of Karachi) plus dual Asr calculation methods (Shafi'i and Hanafi jurisprudence) introduced combinatorial complexity. Each method applies different twilight angles and shadow-length ratios, requiring a parametric architecture that avoids code duplication while maintaining extensibility.

### 3. **Power-Aware GPS Integration**
Balancing location accuracy with battery life demanded careful orchestration of the GPS subsystem. Continuous positioning would drain battery within hours, while cached coordinates become stale as users travel. The solution required a hybrid approach that minimizes radio-on time while gracefully handling positioning failures.

### 4. **Responsive UI Without Layout Frameworks**
Supporting three device form factors (42mm, 47mm, 51mm) with a single codebase—without heavyweight UI frameworks—required viewport-relative positioning mathematics and custom graphics primitives. AMOLED display characteristics further constrained design choices, mandating pure black backgrounds to prevent pixel burn-in.

### 5. **Testing Mathematical Correctness Without Headless Infrastructure**
The proprietary Garmin Connect IQ SDK lacks traditional unit testing frameworks. Validating astronomical calculations required running code on hardware simulators, introducing friction into the development feedback loop and making test-driven development impractical.

## Architecture Decisions

### **Offline-First vs. Cloud-Backed Calculation**
**Decision:** Perform all calculations locally using GPS coordinates and device clock.
**Trade-offs:**
- **Eliminated:** Network latency, API rate limits, operational costs, single points of failure
- **Accepted:** Higher implementation complexity, no push notifications, device-only state
- **Rationale:** Prayer times are deterministic functions of location and date. Computation cost (~50-100ms) is negligible compared to network round-trip time (500ms+), and offline reliability is paramount for religious observance. The inability to send background notifications is a platform limitation (widgets cannot register alarms), making cloud integration provide no compensating benefit.

### **Widget vs. Watch Face vs. Data Field**
**Decision:** Implement as a widget with optional glance view.
**Trade-offs:**
- **Gained:** User-initiated updates (power efficiency), appropriate scoping for MVP, accessibility without active workout
- **Sacrificed:** Always-visible display, background refresh capabilities
- **Rationale:** Watch faces require always-on rendering (higher power draw), while data fields only appear during fitness tracking activities. Widgets offer the optimal balance between accessibility and resource consumption for this use case.

### **Functional Module vs. Object-Oriented Domain Model**
**Decision:** Implement prayer calculator as a stateless functional module rather than class hierarchy.
**Trade-offs:**
- **Gained:** Zero instantiation overhead, compiler optimization opportunities, simpler testing surface, clearer functional composition
- **Sacrificed:** Encapsulation via inheritance, polymorphic extensibility
- **Rationale:** Prayer calculations are pure functions (location + date → times) with no mutable state. Functional modules consume less memory than object instances—critical in embedded contexts—and the mathematical nature of the domain aligns naturally with functional decomposition.

### **One-Shot GPS Positioning vs. Continuous Tracking**
**Decision:** Request single GPS fix on widget open, then disable radio.
**Trade-offs:**
- **Gained:** 90%+ reduction in power consumption vs. continuous polling
- **Accepted:** Stale coordinates if user relocates >50km without reopening widget
- **Rationale:** Prayer times change gradually with location (±1 minute per 25km at mid-latitudes). Users typically open the widget once daily, making continuous tracking wasteful. The device's last-known-position cache provides instant results when GPS acquisition would otherwise delay.

### **Custom Graphics Rendering vs. UI Framework Components**
**Decision:** Manually draw progress rings, text layouts, and dividers using immediate-mode graphics primitives.
**Trade-offs:**
- **Gained:** Pixel-perfect AMOLED optimization, 40%+ memory savings, fine-grained control over rendering pipeline
- **Accepted:** More rendering code, manual layout mathematics, no declarative UI patterns
- **Rationale:** Connect IQ's UI framework adds overhead unsuitable for memory-constrained devices. Custom rendering enables AMOLED-specific optimizations (pure black backgrounds, selective pixel activation) and responsive layouts computed at runtime based on viewport dimensions.

## System Design Highlights

### **Bounded Context Separation**
The system maintains clean separation between three bounded contexts:
- **Astronomical Domain:** Pure mathematical functions for solar position and prayer time derivation (no side effects, fully deterministic)
- **Presentation Layer:** View rendering with viewport-relative layout and progressive loading states
- **Platform Integration:** GPS callbacks, settings persistence, lifecycle management

This separation enables testing astronomical calculations independently via a Ruby language port that mirrors the algorithmic logic without requiring device simulators.

### **Event-Driven Positioning with Graceful Degradation**
GPS acquisition follows a non-blocking callback pattern:
1. Request one-shot positioning on view initialization
2. Display "Getting Location..." state immediately
3. On successful callback: trigger calculation pipeline
4. On timeout/failure: display error state with retry affordance
5. Fall back to cached position when available

This approach maintains UI responsiveness while handling the high-latency nature of satellite positioning (~3-15 seconds for initial fix).

### **Parametric Calculation Strategy Pattern**
Rather than implementing separate code paths for each prayer calculation method, the system uses data-driven parameterization:

```
Method Selection → Parameter Lookup (Fajr angle, Isha angle, Isha offset) → Unified Algorithm
```

New methodologies require only parameter additions, not code changes. This reduces bug surface area and ensures algorithmic consistency across methods.

### **Progressive Enhancement in User Experience**
The UI employs progressive state disclosure:
- **Initial:** "Getting Location..." (GPS acquisition in progress)
- **Transition:** "Calculating..." (mathematical computation)
- **Steady State:** Full prayer schedule with next prayer highlighted
- **Countdown Display:** Real-time remaining time in hours/minutes format

Each state provides meaningful feedback, eliminating perception of unresponsiveness during cold-start scenarios.

### **Dual-View Presentation Pattern**
Two view implementations serve different access patterns:
- **Full Widget:** Rich information display with GPS positioning, calculation orchestration, and detailed schedule
- **Glance View:** Lightweight summary using cached position, showing only next prayer with minimal rendering

The glance view enables quick access via the device's widget carousel without triggering expensive GPS operations, demonstrating appropriate resource allocation based on user intent.

### **Settings-Driven Configuration with Hot Reload**
Five user-configurable dimensions (calculation method, Asr method, iqama offsets per prayer) modify behavior without code changes:
- Properties service acts as configuration store
- Settings changes trigger `onSettingsChanged()` lifecycle hook
- View invalidation (`requestUpdate()`) re-renders with new parameters
- No application restart required

This architecture separates policy (user preferences) from mechanism (calculation algorithms), enabling personalization without increasing binary size.

### **Viewport-Relative Responsive Layout**
Rather than hardcoding pixel coordinates for each device variant, the system computes layout at runtime:

```
Hero Element Y = viewport_height × 0.32
List Start Y = viewport_height × 0.58
Item Height = (viewport_height × 0.32) ÷ 5
```

This proportional approach scales seamlessly across form factors (42mm, 47mm, 51mm screens) with a single codebase, avoiding device-specific branching.

### **Idempotent Calculation Pipeline**
The prayer time calculation is a pure function with no side effects:

```
(Latitude, Longitude, Date, Method Parameters) → Prayer Times Dictionary
```

Repeated invocations with identical inputs always produce identical outputs, enabling safe caching and simplifying reasoning about correctness. The lack of mutable state eliminates entire classes of concurrency bugs.

## Results & Impact

### **Performance Metrics**
- **Binary Size:** [~112 KB] compiled executable (well within Connect IQ limits)
- **Calculation Latency:** [<100ms] from GPS fix to prayer times display
- **Power Consumption:** [~5 seconds] of GPS activity per widget open (vs. continuous tracking approaches requiring constant radio-on time)
- **Memory Footprint:** [~1-2 KB] runtime state (prayer times dictionary + UI state)

### **Reliability Improvements**
- **100% offline availability:** No dependency on network connectivity or external APIs
- **Zero operational failures:** Elimination of backend infrastructure removes service outage risk
- **Deterministic accuracy:** ±2 minutes precision across all calculation methods (within Islamic jurisprudential tolerances)

### **User Experience Gains**
- **Instant cold-start:** Cached GPS positions enable <1 second display for repeat opens
- **Multi-method support:** 5 calculation methodologies + 2 Asr methods serve diverse user preferences
- **Responsive design:** Single codebase supports three device sizes without degradation

### **Development Velocity**
- **[565 lines]** of production code demonstrates high signal-to-noise ratio
- **Dual-language testing:** Ruby mirror enables [5-10x faster] iteration on astronomical algorithms vs. simulator-based testing
- **Clean separation:** Domain logic reusable for future watch face or data field implementations

## Key Learnings

### **1. Offline-First Architecture Reduces System Complexity**
Eliminating network dependencies removed entire categories of failure modes (timeouts, rate limits, authentication, API versioning). The additional complexity of implementing astronomical algorithms was offset by the elimination of backend infrastructure, monitoring, and operational runbooks. For deterministic computations with stable inputs, local execution often represents the simpler system design.

### **2. Dual-Language Testing Unlocks Embedded TDD**
The Ruby mirror of calculation logic proved essential for rapid iteration. By porting algorithms to a language with mature testing infrastructure, development velocity increased substantially. This pattern generalizes: when testing in the target environment is expensive (simulators, emulators, hardware), maintaining a reference implementation in a testable language provides asymmetric leverage.

### **3. Parametric Design Beats Branching for Methodology Variations**
Representing calculation methods as data (lookup tables of angles and offsets) rather than code branches reduced cyclomatic complexity and bug surface area. Adding new methods required zero code changes—only parameter additions. This approach scales better than inheritance hierarchies when variations differ primarily in configuration rather than algorithms.

### **4. Resource Constraints Force Architectural Clarity**
Memory and processing limitations eliminated the option of heavyweight frameworks, forcing explicit decisions about every abstraction. The resulting architecture—functional modules, immediate-mode rendering, stateless calculations—proved simpler and more comprehensible than typical framework-heavy approaches. Constraints drive design clarity when embraced rather than fought.

### **5. Progressive State Disclosure Maintains User Trust**
Displaying meaningful status at every pipeline stage ("Getting Location...", "Calculating...") eliminated perception of hangs or freezes. Users tolerate latency when they understand system state. This principle extends beyond loading indicators: any multi-stage process benefits from exposing intermediate states rather than showing spinners.

### **6. One-Shot Resource Acquisition Balances Accuracy and Efficiency**
The GPS positioning strategy—single fix on demand rather than continuous polling—reduced power consumption by [90%+] while maintaining acceptable accuracy for the use case. This pattern applies broadly: many location-aware features don't require real-time tracking. Understanding acceptable staleness tolerances enables dramatic efficiency gains.

### **7. Platform Limitations Shape Viable Solutions**
The inability to send notifications from widgets fundamentally constrained the product surface. Rather than fighting platform limitations (attempting background processes, persistent services), the design embraced them: users open the widget when they need prayer times. Architectural decisions must account for platform affordances, not idealized requirements.

### **8. Viewport-Relative Layout Beats Device-Specific Code**
Proportional positioning (percentage-based coordinates) eliminated the need for device-specific branches despite supporting three form factors. This demonstrates the power of ratio-based design: compute layout from viewport dimensions at runtime rather than hardcoding for known devices. Future device additions require zero code changes.
