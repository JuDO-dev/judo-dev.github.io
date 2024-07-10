@def title = "JuDO - Julia for Dynamic Optimization"
@def tags = ["syntax", "code"]

Solve dynamic optimization problems in a quick and intuitive way.

# As easy as one-two-three

## 1. Choose a solver

```julia
using JuDO, Interesso
model = Model(Interesso.Optimizer)
```

## 2. Define your problem

```julia
# Time
@phase(model, t)
@constraint(model, initial(t) == 0.0)
@constraint(model, final(t) == 2.0)

# States
@dynamic_variable(model, r, t)
@dynamic_variable(model, v, t)
@dynamic_variable(model, θ, t)
@dynamic_variable(model, ω, t)

# Input
@dynamic_variable(model, -20.0 ≤ u ≤ 20.0, t)

# Differential Equations
@constraint(model, ∂(r) == v)
@constraint(model, ∂(v) == ...)
@constraint(model, ∂(θ) == ω)
@constraint(model, ∂(ω) == ...)

# Boundary Conditions
@constraint(model, initial(r) == 0.0)
@constraint(model, initial(v) == 0.0)
@constraint(model, initial(θ) == 0.0)
@constraint(model, initial(ω) == 0.0)

@constraint(model, final(r) == 2.0)
@constraint(model, final(v) == pi)
@constraint(model, final(θ) == 0.0)
@constraint(model, final(ω) == 0.0)

# Objective Function
@objective(model, Min, ∫(u^2))
```

## 3. Get a solution

```julia
optimize!(model)
plot(r)
```