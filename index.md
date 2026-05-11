@def title = "JuDO - Julia for Dynamic Optimization"
@def tags = ["syntax", "code"]
@def hascode = true
@def hasmath = true

**JuDO.jl** is a Julia package for formulating and solving dynamic optimisation problems (optimal control, trajectory optimisation, and more). It extends the [JuMP](https://jump.dev) modelling language with continuous-time variables, differential equations, and integral objectives — keeping solver details out of your model code.

# As easy as one-two-three

## 1. Install

JuDO and its dependencies are under active development and not yet registered in the Julia General registry. Install directly from GitHub:

```julia
using Pkg
Pkg.add(url="https://github.com/shawn-tao01/DynOptInterface.jl", rev="dev")
Pkg.add(url="https://github.com/Kailai-Shi/Interesso.jl",        rev="dev")
Pkg.add(url="https://github.com/shawn-tao01/JuDO.jl", rev="dev")
```

Julia 1.12 or later is required.

## 2. Define your problem

The example below solves the classic **cart-pole swing-up**: a pendulum on a cart must be swung from hanging ($\theta=0$) to upright ($\theta=\pi$) by a horizontal force, while minimising total control effort $\int_0^{t_f} u^2 \, dt$.

```julia
using JuDO, Interesso

const g = 9.81;  const l = 0.5;  const m_1 = 1.0;  const m_2 = 0.3
const u_max = 20.0;  const r_max = 2.0

dop = DynModel(Interesso.Optimizer)

# Independent variable (time phase)
@phase(dop, t)
@constraint(dop, initial(t) == 0)
@constraint(dop, final(t)   == 2)

# States
@variable(dop, 0 <= r <= r_max, DefinedOn(t))   # cart position (m)
@variable(dop, ν,               DefinedOn(t))   # cart velocity (m/s)
@variable(dop, θ,               DefinedOn(t))   # pole angle (rad)
@variable(dop, ω,               DefinedOn(t))   # pole angular velocity (rad/s)

# Control
@variable(dop, -u_max <= u <= u_max, DefinedOn(t))  # horizontal force (N)

# Boundary conditions: hanging → upright
@constraint(dop, initial(r) == 0);  @constraint(dop, final(r) == 1)
@constraint(dop, initial(ν) == 0);  @constraint(dop, final(ν) == 0)
@constraint(dop, initial(θ) == 0);  @constraint(dop, final(θ) == pi)
@constraint(dop, initial(ω) == 0);  @constraint(dop, final(ω) == 0)

# Equations of motion (Lagrangian dynamics)
@constraint(dop, derivative(r) == ν)
@constraint(dop, derivative(ν) == (l*m_2*sin(θ)*ω^2 + u + m_2*g*cos(θ)*sin(θ)) /
                                  (m_1 + m_2*sin(θ)^2))
@constraint(dop, derivative(θ) == ω)
@constraint(dop, derivative(ω) == (-l*m_2*cos(θ)*sin(θ)*ω^2 - u*cos(θ) - (m_1 + m_2)*g*sin(θ)) /
                                  (l*(m_1 + m_2*sin(θ)^2)))

# Minimise control effort
@objective(dop, Min, integral(u^2))
```

## 3. Solve and extract the solution

```julia
using DynOptInterface

# Warm-start with linear guesses between boundary values
struct LinearInterpolant <: DynOptInterface.AbstractDynamicSolution
    y_a::Float64;  y_b::Float64
end
(li::LinearInterpolant)(t::Real) = li.y_a + t * (li.y_b - li.y_a) / 2.0

JuDO.warmstart!(dop, LinearInterpolant(0.0, 1.0), r)
JuDO.warmstart!(dop, LinearInterpolant(0.0, pi),  θ)

JuDO.optimize!(dop)

# Solution trajectories are callable functions of time
r_sol = dyn_value(dop, r)
θ_sol = dyn_value(dop, θ)
u_sol = dyn_value(dop, u)

using Plots
ts = range(0, 2; length=200)
plot(ts, θ_sol.(ts); xlabel="Time (s)", ylabel="Pole angle θ (rad)", legend=false)
```

See the [Examples](/Examples/) page for the full annotated code and the aerospace [Space Shuttle Reentry](/Examples/#space_shuttle_reentry) benchmark (maximising crossrange; reference solution ≈ 34.14°).

---

# The JuDO ecosystem

JuDO follows the same layered architecture as JuMP / MathOptInterface:

| Layer | Package | Role |
|-------|---------|------|
| **Modelling** | [JuDO.jl](https://github.com/JuDO-dev/JuDO.jl) | Solver-agnostic problem formulation |
| **Interface** | [DynOptInterface.jl](https://github.com/shawn-tao01/DynOptInterface.jl) | Standard intermediate representation for dynamic optimisation |
| **Solver** | [Interesso.jl](https://github.com/Kailai-Shi/Interesso.jl) | Collocation-based NLP transcription via Ipopt |

A problem written in JuDO can in principle be solved by any solver that implements the DynOptInterface (DOI) standard — with no changes to the model code.
See the [Packages](/Packages/) page for details on each package.
