@def title = "JuDO - Julia for Dynamic Optimization"
@def tags = ["syntax", "code"]
@def hascode = true
@def hasmath = true

# About JuDO

**JuDO** (Julia for Dynamic optimization) is an open-source modeling language developing a modern, solver-agnostic ecosystem for packages for dynamic optimization in Julia.

Dynamic optimization problems arises across engineering fields (spacecraft trajectory design, robot motion planning, process control, etc.). 
JuDO provides a simple formulation layer for users to define their problems mathematically, and an underlying abstracton layer DynOptInterface that provides necessary transformations to fit the dynamic-optimization solver.

---

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

The example below solves a classic LQR problem formulation.

```julia
using Interesso, JuMP, JuDO, DynOptInterface
using Plots


model = DynModel(Interesso.Optimizer)

@phase(model, t, 0.0, 10.0)

## Dynamic Variables
@variable(model, -1.0 <= u <= 1.0, DefinedOn(t))
@variable(model, x, DefinedOn(t))
@variable(model, v, DefinedOn(t))

## Boundary Conditions
@constraint(model, initial(x) == 0.0)
@constraint(model, initial(v) == 0.0)
@constraint(model, final(x) == 20.0)
@constraint(model, final(v) == 0.0)

## Differential Equations
@constraint(model, derivative(v) == u)
@constraint(model, derivative(x) == v)

## Objective
@objective(model, Min, integral(2*x^2 + 2*v^2))

JuDO.optimize!(model)

## Extract solutions
t_0 = phase_initial(t).value
t_f = 10.0

x_sol = dyn_value(model, x)
v_sol = dyn_value(model, v)
u_sol = dyn_value(model, u)

## Plot
p1 = plot(τ -> x_sol(τ), t_0, t_f; label="x(t)", ylabel="Position")
p2 = plot(τ -> v_sol(τ), t_0, t_f; label="v(t)", ylabel="Velocity")
p3 = plot(τ -> u_sol(τ), t_0, t_f; label="u(t)", ylabel="Control", xlabel="t")

display(plot(p1, p2, p3; layout=(3, 1), size=(600, 700), legend=:topright))
```
