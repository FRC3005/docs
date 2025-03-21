# 2022-2023 Telemetry Rewrite Proposal

This is a design document for a proposed refactoring of WPILib telemetry features for the 2022-2023 season.

## Motivations

As WPILib has grown, its telemetry architecture has grown with it.  Heading into the 2022-2023 season, WPILib currently plans to support:

* A communications protocol (NetworkTables)
* Four dashboards (LabVIEW dashboard, SmartDashboard, Shuffleboard, Glass)
* Two logging solutions (Shuffleboard recordings, on-RIO logging)
* A declarative object-oriented telemetry API (Sendable)
* A code-bypassing "debug mode" for sensors and actuators (LiveWindow)

While the WPILib telemetry ecosystem is reasonably functional (especially when combined with some amount of external tooling), it can be confusing from the outside due to the large amount of duplication in the above tools ("which of the four dashboards should I use and why?").  Additionally, as a result of having emerged from a shared set of "ancestral" features, many of the above tools have overlapping implementations that interact in surprising and sometimes undocumented ways.

Healthy software development generally follows two alternating phases: expansion and consolidation.  The WPI telemetry ecosystem has grown through steady expansion into a substantial and powerful set of features.  It is now in need of a consolidation phase to reduce the total API surface area.  Consolidating and refactoring the existing implementations will serve two purposes: it will provide new users with a clearer and more-intuitive "happy path" for their robot telemetry, and it will reduce the maintenance burden on on the WPILib development team by disentangling the implementations of unrelated API features and reducing the total volume of maintained code.

### Too many dashboards

The current API presents new users with a number of pain points.  Firstly, users are presented with a choice of four different dashboards - each with slightly-different featuresets, but without enough distinguishing features to clearly guide users as to which dashboard is useful in which situation.

In particular:

* The LabVIEW dashboard and SmartDashboard are largely legacy features that have been totally supplanted in functionality by Shuffleboard and Glass.  Neither is likely to be easily maintainable in the event of a major breakage.  Both options are substantially clunkier and less-functional than the alternatives, and teams which choose these dashboards are probably worse-off for the choice.  Is continued support for legacy team-side dashboard code really worth this cost, especially given the tendency of FRC teams to rewrite most of their code anually?
* ShuffleBoard is clearly superior to Glass as an operating dashboard, and has many features (robot-code-driven tabs and layouts, data recording and playback) that make it appealing as the "default" robot dashboard.  However, it is already in danger of slipping into the "legacy code" state of the above two tools due to the migration of developer effort to Glass.  This could eventually cost teams if support for the dashboard becomes difficult to sustain while no other dashboard option offers a similar featureset.
* Glass is a *fantastic* tool for real-time plotting and signal analysis, but currently lacks a robust featureset to be usable as a dashboard.

Under the current ecosystem, effective users will likely want to make use of both Shuffleboard and Glass, while ignoring the LabVIEW dashboard and SmartDashboard.  However, the documentation and API shapes do not necessarily lead users to this conclusion.  Moreover, developer effort is difficult to split between two tools, but the current distribution of features between tools demands it.

### Overloading of the Sendable implementation

On the surface, the `Sendable` interface offers a powerful abstraction to teams who wish to log their state declaratively rather than imperatively.  However, behind-the-scenes `Sendable` pulls double-duty as the backbone of the `LiveWindow` implementation, which is responsible for auto-populating a list of sensors and actuators for easy debugging.

The `LiveWindow` design philosophy holds that all sensor and actuator objects in code should be accessible via the dashboard while the robot is in Test Mode, *regardless of what the user has done in their robot code.*  While this is a nice idea in theory, it places *huge* constraints on the handling of the `Sendable` interface which often interfere with the latter's use as a telemetry handler:

* While writes to the dashboard work as expected, reads from the dashboard (i.e. calls to the bound setters) are suspended *except* in LiveWindow mode.  This is not documented anywhere on the class, and is extremely inconvenient.
* The API includes an `isActuator` flag which seemingly serves no purpose other than to disable certain widgets on ShuffleBoard (it does not interact with the other dashboards).
* Sendables can be registered with `LiveWindow` even if they have no business being there (i.e. they are not a sensor or an actuator).
* It is not documented which `Sendable`s automatically add themselves to the `LiveWindow` registry immediately upon construction, and the application of the `Sendable` interface to certain classes which would otherwise be able to provide useful telemetry to teams (e.g. the various pneumatics classes) has been delayed or abandoned due to the complexities of maintaining a such a static registry, even when the classes could feasibly implement the interface and expose themselves to users sans automatic registration.
  
### Crude interaction between Sendable and Commands

Parts of the `Sendable` interface - particulary those involving the `LiveWindow` registry - have hard dependencies on the design philosophy of the Command-based framework.  In particular, `LiveWindow` widgets are grouped by "subsystem," even though they are populated automatically on robots that may not have recognizable `Subsystem` classes.  The relevant values are silently defaulted - and the functionality rendered useless - in all but a very narrow set of circumstances.

The command scheduler implements `NTSendable` in a fairly ad hoc way (the functionality was copied directly from the original command framework to allow backwards compatibility, which may have been a mistake).  This is one of the few major reliances on `NTSendable` remaining in the codebase, and may present an opportunity to remove the dependency entirely.

### Limited Sendable composability

While the NT architecture supports hierarchical structures (albeit indirectly through parsing of key names), the `Sendable` API only allows *flat* composition (excluding the aforementioned `Subsytem` tagging in the `LiveWindow` registry, which most teams are probably not aware of for the reasons mentioned in the previous section).  That is, child objects that implement `Sendable` can *only* be made to place their logged fields directly in the table entry of their parent.  This makes it very hard to scale `Sendable` in a sensible way, and can cause namespace collisions even in projects small enough to avoid the scaling issues.

### Verbose Shuffleboard API

The Shuffleboard API is extremely powerful and allows more or less the entire dashboard layout to be declaratively structured from robot code.  The native API for this, however, is verbose and not self-documenting.  Simple declarations require long method chains, and widget arguments are passed via string key-value pairs without type safety.

### Lack of a universal metadata layer

NT provides a great communication layer for values, but it doesn't provide any way to provide metadata about those values (eg: units, display parameters, logging frequency).

The implementation of the code-driven layout configuration in Shuffleboard involves an ad-hoc formatting metadata layer that is not shared by any of the other tooling, and so cannot presently be leveraged to do the above.

### Obscure calls for simple functions

The simplest way to log telemetry to the dashboard declaratively is through a `SmartDashboard` method, despite `SmartDashboard` being a nearly-obsolete legacy framework with no active maintenance.  As Glass does not have its own API, the `Shuffleboard` API is rather verbose, `SmartDashboard` calls are often used even in codebases that never actually open `SmartDashboard`.  This is not ideal.
