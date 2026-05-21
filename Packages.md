@def title = "Packages"
@def hascode = true
@def hasmath = true
@def tags = ["packages"]

# JuDO Packages

The JuDO ecosystem consists of two main packages arranged in a layered architecture.

---

## Design philosophy

JuDO has a layered architecture modelled on JuMP / MathOptInterface:

```
User code  (JuDO macros — @phase, @variable, @constraint, @objective...)
                            │
                            ▼
                        JuDO.jl
                            │
                            ▼
                DynOptInterface.jl (DOI)
               /                        \
              /                          \
             ▼                            ▼
    Interesso.jl                 Other DOI-compatible
          │                           solvers...
          ▼
   NLP solver (Ipopt)
```

---

## JuDO.jl

**The user-facing modelling layer.**

JuDO.jl extends [JuMP](https://jump.dev) with constructs for dynamic optimization: time phases, trajectory variables, derivatives, boundary operators, and integral objectives. Problems are formulated in a solver-agnostic syntax and translated automatically into the DynOptInterface (DOI) standard representation.

Key modelling constructs:

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
