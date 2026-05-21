@def title = "Packages"
@def hascode = true
@def hasmath = true
@def tags = ["packages"]

# JuDO Packages

The JuDO ecosystem consists of two main packages arranged in a layered architecture.

---

## Design philosophy

JuDO has a layered architecture:

~~~
<svg viewBox="0 0 700 440" xmlns="http://www.w3.org/2000/svg" style="max-width:100%; display:block; margin:2rem auto; font-family:system-ui,sans-serif;">

  <!-- User Code block -->
  <rect x="100" y="10" width="500" height="65" rx="8" fill="#4f46e5"/>
  <text x="350" y="33" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="15">User Code</text>
  <text x="350" y="53" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.85)" font-size="12" font-family="monospace">@phase  @variable  @constraint  @objective</text>

  <!-- Arrow: User Code → JuDO.jl -->
  <line x1="350" y1="75" x2="350" y2="102" stroke="#6b7280" stroke-width="2"/>
  <polygon points="350,105 344,95 356,95" fill="#399ec3"/>

  <!-- JuDO.jl block -->
  <rect x="200" y="105" width="300" height="55" rx="8" fill="#166ba3"/>
  <text x="350" y="123" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="15">JuDO.jl</text>
  <text x="350" y="143" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.85)" font-size="12">Modelling layer</text>

  <!-- Arrow: JuDO.jl → DOI -->
  <line x1="350" y1="160" x2="350" y2="187" stroke="#6b7280" stroke-width="2"/>
  <polygon points="350,190 344,180 356,180" fill="#6b7280"/>

  <!-- DynOptInterface.jl (DOI) block -->
  <rect x="150" y="190" width="400" height="55" rx="8" fill="#0606d9"/>
  <text x="350" y="208" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="15">DynOptInterface.jl (DOI)</text>
  <text x="350" y="228" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.85)" font-size="12">Abstraction layer</text>

  <!-- Fork from DOI bottom: 3-way split -->
  <line x1="350" y1="245" x2="350" y2="265" stroke="#6b7280" stroke-width="2"/>
  <line x1="123" y1="265" x2="578" y2="265" stroke="#6b7280" stroke-width="2"/>
  <!-- Left branch to Interesso -->
  <line x1="123" y1="265" x2="123" y2="287" stroke="#6b7280" stroke-width="2"/>
  <polygon points="123,290 117,280 129,280" fill="#6b7280"/>
  <!-- Middle branch to CTDirect -->
  <line x1="350" y1="265" x2="350" y2="287" stroke="#6b7280" stroke-width="2"/>
  <polygon points="350,290 344,280 356,280" fill="#6b7280"/>
  <!-- Right branch to Other solvers (dashed to suggest extensibility) -->
  <line x1="578" y1="265" x2="578" y2="287" stroke="#6b7280" stroke-width="2" stroke-dasharray="5,3"/>
  <polygon points="578,290 572,280 584,280" fill="#6b7280"/>

  <!-- Interesso.jl block -->
  <rect x="20" y="290" width="207" height="55" rx="8" fill="#2f4b5e"/>
  <text x="123" y="308" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="14">Interesso.jl</text>
  <text x="123" y="328" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.85)" font-size="11">Integrated Residuals solver</text>

  <!-- CTDirect.jl block -->
  <rect x="247" y="290" width="207" height="55" rx="8" fill="#0369a1"/>
  <text x="350" y="308" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="14">CTDirect.jl</text>
  <text x="350" y="328" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.85)" font-size="11">Direct collocation solver</text>

  <!-- Other DOI-compatible solvers block -->
  <rect x="474" y="290" width="207" height="55" rx="8" fill="#6b7280"/>
  <text x="578" y="308" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="13">Other DOI-compatible</text>
  <text x="578" y="328" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.9)" font-size="11">solvers...</text>

  <!-- Arrow: Interesso.jl → NLP Solver -->
  <line x1="123" y1="345" x2="123" y2="372" stroke="#6b7280" stroke-width="2"/>
  <polygon points="123,375 117,365 129,365" fill="#6b7280"/>

  <!-- NLP Solver (Ipopt) block -->
  <rect x="20" y="375" width="207" height="50" rx="8" fill="#374151"/>
  <text x="123" y="390" dominant-baseline="middle" text-anchor="middle" fill="white" font-weight="bold" font-size="14">NLP Solver</text>
  <text x="123" y="410" dominant-baseline="middle" text-anchor="middle" fill="rgba(255,255,255,0.85)" font-size="12">via Ipopt</text>

</svg>
~~~

---

## JuDO.jl

**The user-facing modelling layer.**

JuDO.jl extends [JuMP](https://jump.dev) with features for dynamic optimization. Problems are formulated in a solver-agnostic syntax and translated automatically into the DynOptInterface (DOI) standard representation.

Key modelling features:

| Macro / Function | Purpose |
|-----------------|---------|
| `DynModel(optimizer)` | Create a dynamic optimization model |
| `@phase(model, t)` | Declare the independent variable (time) |
| `@variable(model, bounds, DefinedOn(t))` | Declare a trajectory variable |
| `initial(x)`, `final(x)` | Access trajectory endpoints in constraints |
| `derivative(x)` | Declare the time derivative $\dot{x}(t)$ |
| `@objective(model, sense, integral(expr))` | Lagrange (running) objective |
| `@objective(model, sense, final(expr))` | Mayer (terminal) objective |
| `JuDO.warmstart!(model, interpolant, x)` | Provide an initial guess trajectory |
| `JuDO.optimize!(model)` | Solve the problem |
| `dyn_value(model, x)` | Extract solution as a callable `t -> value` |

**[Source Code](https://github.com/JuDO-dev/JuDO.jl)**  
**[Documentation](https://JuDO-dev.github.io/JuDO.jl/dev/)**  

---

## DynOptInterface.jl

**The mathematical abstraction layer (DOI).**

DynOptInterface.jl (DOI) is to dynamic optimization what MathOptInterface (MOI) is to static mathematical programming. It defines a standard intermediate representation for Dynamic optimization Problems (DOPs) that solvers can attach to.

The general problem form DOI represents:

$$
\min_{x,\,y(\cdot)} \quad f_o(x) + b_o\!\left(y(t_0), y(t_f), t_0, t_f, x\right) + \sum_i \int_{t_0^{(i)}}^{t_f^{(i)}} d_o\!\left(\dot y(t), y(t), t, x\right) \mathrm{d}t
$$

subject to finite-dimensional constraints $f_c(x) \in S$, boundary constraints $b_c(\cdot) \in B$, and dynamic/path constraints $d_c(\dot y, y, t, x) \in D$ on each phase $[t_0^{(i)}, t_f^{(i)}]$.

Key abstractions:

| Type | Purpose |
|------|---------|
| `Phase` | A time interval $[t_0, t_f]$ with variable boundaries |
| `DynamicVariable` | An optimizable trajectory $y(t)$ |
| `Derivative` | Time derivative $\dot y(t)$ |
| `Initial`, `Final` | Boundary-value operators |
| `Integral`, `MultiPhaseIntegral` | Lagrange cost terms |
| `Bolza` | Combined Mayer + Lagrange objective |

**[Source Code](https://github.com/shawn-tao01/DynOptInterface.jl)**  
**[Documentation](https://judo.dev/DynOptInterface.jl/dev/)**  

**Current version: v0.3.0** 

---

## Interesso.jl

**The dynamic optimization solver.**

Interesso.jl is a solver we develop that could attach to the DynOptInterface. It transcribes continuous-time DOPs into large-scale Nonlinear Programs (NLPs) using an **Integrated Residuals Method** (IRM) and dispatches them to [Ipopt](https://coin-or.github.io/Ipopt/).

Apart from Interesso, we have also interfaced CTDirect.jl in [OptimalControl](https://github.com/control-toolbox/OptimalControl.jl) toolbox under DOI, which demonstrates the capability of attaching existing solvers under DOI.

Features:
- Multi-phase dynamic optimization
- Explicit and implicit ODE dynamics
- Free final time
- NLP warm-starting via `JuDO.warmstart!`
- Post-solve trajectory interpolants (callable solution functions)
- Plots.jl extension for solution visualisation

**Example problems included:**

| Example | Description |
|---------|-------------|
| Cart-pole swing-up | Classic pendulum-on-cart control |
| Space shuttle reentry | 6-state aerospace trajectory (benchmark: 34.14° crossrange) |
| Van der Pol oscillator | Nonlinear oscillator |
| Double integrator | LQR-type minimum-effort control |
| Bang-bang control | Switching control |
| Orbit raising | Orbital mechanics |
| Fuller's problem | Singular arc |
| Hyper-sensitive | Stiff boundary layer |
| Two-link robot arm | Robotics trajectory |
| Linear bicycle model | Vehicle dynamics |

**[Source Code](https://github.com/Kailai-Shi/Interesso.jl)**  
---

Julia 1.12 or later is required.
