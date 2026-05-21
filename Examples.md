@def title = "Examples"
@def hascode = true
@def hasmath = true
@def tags = ["examples", "tutorials"]

# Examples

This page contains complete, runnable JuDO examples. Each example uses:
- `DynModel(Interesso.Optimizer)` to create a model backed by the Interesso solver
- JuDO macros (`@phase`, `@variable`, `@constraint`, `@objective`) for problem formulation
- `JuDO.warmstart!` to provide an initial guess
- `dyn_value(model, x)` to extract solution trajectories as callable functions

### Shared setup

Both examples below use a simple linear warm-start interpolant. Define it once per Julia session before running either example:

```julia
using JuDO, Interesso, DynOptInterface
using Plots

struct LinearInterpolant <: DynOptInterface.AbstractDynamicSolution
    y_a::Float64
    y_b::Float64
    t_0::Float64
    t_f::Float64
end
(li::LinearInterpolant)(t::Real) = li.y_a + (t - li.t_0) * (li.y_b - li.y_a) / (li.t_f - li.t_0)
```

---

## Cart-Pole Swing-Up

The cart-pole swing-up is a classic control benchmark. A pole is attached via a pivot to a cart that moves along a frictionless track. Starting from the hanging position ($\theta = 0$), the goal is to swing the pole to the upright position ($\theta = \pi$) by applying a horizontal force to the cart, while minimising total control effort.

### Problem formulation

**Parameters**

| Symbol | Value | Description |
|--------|-------|-------------|
| $m_1$ | 1.0 kg | Cart mass |
| $m_2$ | 0.3 kg | Pole mass |
| $l$   | 0.5 m  | Pole half-length |
| $g$   | 9.81 m/s² | Gravitational acceleration |
| $u_{\max}$ | 20 N | Maximum control force |
| $r_{\max}$ | 2 m  | Maximum cart displacement |

**Optimal control problem**

$$
\min_{u(\cdot)} \quad \int_0^{2} u(t)^2 \, \mathrm{d}t
$$

subject to the Lagrangian equations of motion:

$$
\dot r = \nu, \qquad
\dot \nu = \frac{l \, m_2 \sin\theta \cdot \omega^2 + u + m_2 g \cos\theta \sin\theta}{m_1 + m_2 \sin^2\theta}
$$

$$
\dot \theta = \omega, \qquad
\dot \omega = \frac{-l \, m_2 \cos\theta \sin\theta \cdot \omega^2 - u \cos\theta - (m_1 + m_2) g \sin\theta}{l (m_1 + m_2 \sin^2\theta)}
$$

with boundary conditions $r(0)=\nu(0)=\theta(0)=\omega(0)=0$ and $r(2)=1,\;\nu(2)=0,\;\theta(2)=\pi,\;\omega(2)=0$.

### JuDO code

```julia
using JuDO, Interesso, DynOptInterface
using Plots

const g     = 9.81
const l     = 0.5
const m_1   = 1.0
const m_2   = 0.3
const t_0   = 0.0
const t_f   = 2.0
const u_max = 20.0
const r_max = 2.0

dop = DynModel(Interesso.Optimizer)

# Independent variable (time)
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

# Boundary conditions
@constraint(dop, initial(r) == 0);  @constraint(dop, final(r) == 1)
@constraint(dop, initial(ν) == 0);  @constraint(dop, final(ν) == 0)
@constraint(dop, initial(θ) == 0);  @constraint(dop, final(θ) == pi)
@constraint(dop, initial(ω) == 0);  @constraint(dop, final(ω) == 0)

# Equations of motion
@constraint(dop, derivative(r) == ν)
@constraint(dop, derivative(ν) == (l*m_2*sin(θ)*ω^2 + u + m_2*g*cos(θ)*sin(θ)) /
                                  (m_1 + m_2*sin(θ)^2))
@constraint(dop, derivative(θ) == ω)
@constraint(dop, derivative(ω) == (-l*m_2*cos(θ)*sin(θ)*ω^2 - u*cos(θ) - (m_1 + m_2)*g*sin(θ)) /
                                  (l*(m_1 + m_2*sin(θ)^2)))

# Minimise control effort
@objective(dop, Min, integral(u^2))

# Warm-start with linear guesses
JuDO.warmstart!(dop, LinearInterpolant(0.0, 1.0, t_0, t_f), r)
JuDO.warmstart!(dop, LinearInterpolant(0.0, pi,  t_0, t_f), θ)

JuDO.optimize!(dop)

# Extract solution trajectories (each is a callable t -> value)
r_sol = dyn_value(dop, r)
θ_sol = dyn_value(dop, θ)
ν_sol = dyn_value(dop, ν)
ω_sol = dyn_value(dop, ω)
u_sol = dyn_value(dop, u)

ts = collect(range(t_0, t_f; length=200))

# Plot pole angle trajectory
p1 = plot(ts, θ_sol.(ts);
    xlabel="Time (s)", ylabel="Pole angle θ (rad)",
    legend=false, lw=1.5, lc=:black)

# Plot cart position trajectory
p2 = plot(ts, r_sol.(ts);
    xlabel="Time (s)", ylabel="Cart position r (m)",
    legend=false, lw=1.5, lc=:black)

# Plot control force
p3 = plot(ts, u_sol.(ts);
    xlabel="Time (s)", ylabel="Control force u (N)",
    legend=false, lw=1.5, lc=:black)

display(plot(p1, p2, p3; layout=(3,1), size=(600, 700)))
```

---

## Space Shuttle Reentry \{#space_shuttle_reentry\}

This benchmark problem, optimizes the reentry trajectory of a space shuttle to **maximise the crossrange** (final latitude $\theta(t_f)$), subject to full six-state atmospheric flight dynamics and terminal boundary conditions. The reference optimal crossrange is approximately **34.14°**.

### Problem formulation

**Physical constants**

| Symbol | Value | Description |
|--------|-------|-------------|
| $m$    | 203000/32.174 slug | Shuttle mass |
| $S$    | 2690 ft² | Reference area |
| $R_e$  | 20902900 ft | Earth radius |
| $\mu$  | 0.14076539 × 10¹⁷ ft³/s² | Gravitational parameter |
| $\rho_0$ | 0.002378 slug/ft³ | Sea-level atmospheric density |
| $h_r$  | 23800 ft | Density scale height |

**Aerodynamic model**

$$
C_L(\alpha) = a_0 + a_1 \alpha_{\deg}, \qquad C_D(\alpha) = b_0 + b_1 \alpha_{\deg} + b_2 \alpha_{\deg}^2
$$

where $\alpha_{\deg} = (180/\pi)\,\alpha$ and $(a_0, a_1) = (-0.20704,\;0.029244)$, $(b_0, b_1, b_2) = (0.07854,\;-0.61592 \times 10^{-2},\;0.621408 \times 10^{-3})$.

**Optimal control problem**

$$
\max_{\alpha(\cdot),\,\beta(\cdot)} \quad \theta(t_f)
$$

subject to the six-state 3-DOF equations of motion over a spherical Earth:

$$
\dot h = v \sin\gamma, \qquad
\dot\theta = \frac{v \cos\gamma \cos\psi}{r}, \qquad
\dot\Phi = \frac{v \cos\gamma \sin\psi}{r \cos\theta}
$$

$$
\dot v = -\frac{D}{m} - g\sin\gamma, \qquad
\dot\gamma = \frac{L\cos\beta}{mv} + \cos\gamma\!\left(\frac{v}{r} - \frac{g}{v}\right), \qquad
\dot\psi = \frac{L\sin\beta}{mv\cos\gamma} + \frac{v\cos\gamma\sin\psi\tan\theta}{r}
$$

where $r = R_e + h$, $g = \mu/r^2$, $\rho = \rho_0 e^{-h/h_r}$, $L = \tfrac{1}{2}S\rho v^2 C_L$, $D = \tfrac{1}{2}S\rho v^2 C_D$.

**Boundary conditions**

| Variable | Initial | Final |
|----------|---------|-------|
| $h$ | 260 000 ft | 80 000 ft |
| $v$ | 25 600 ft/s | 2 500 ft/s |
| $\theta$ | 0 | free (maximised) |
| $\Phi$ | 0 | free |
| $\gamma$ | −1° | −5° |
| $\psi$ | 90° | free |
| $t$ | 0 | $\leq$ 2500 s |

### JuDO code

```julia
using JuDO, Interesso, DynOptInterface
using Plots

dop = DynModel(Interesso.Optimizer)

# Physical constants
const m             = 203000 / 32.174
const ρ_0, h_r, R_e = 0.002378, 23800.0, 20902900.0
const μ             = 0.14076539e17
const a_0, a_1      = -0.20704, 0.029244
const b_0, b_1, b_2 = 0.07854, -0.61592e-2, 0.621408e-3
const S             = 2690

# Warm-start interpolant
struct LinearInterpolant <: DynOptInterface.AbstractDynamicSolution
    y_a::Float64;  y_b::Float64
end
(li::LinearInterpolant)(t::Real) = li.y_a + t * (li.y_b - li.y_a) / 2500

# Independent variable (time)
@phase(dop, t)
@constraint(dop, initial(t) == 0)
@constraint(dop,   final(t) ≤ 2500)

# States
@variable(dop,              0 ≤ h,              DefinedOn(t))  # altitude (ft)
@variable(dop, deg2rad(-89) ≤ θ ≤ deg2rad(89),  DefinedOn(t))  # latitude (rad)
@variable(dop,               Φ,                 DefinedOn(t))  # longitude (rad)
@variable(dop,           1e-4 ≤ v,              DefinedOn(t))  # velocity (ft/s)
@variable(dop, deg2rad(-89) ≤ γ ≤ deg2rad(89),  DefinedOn(t))  # flight path angle (rad)
@variable(dop,               ψ,                 DefinedOn(t))  # azimuth (rad)

# Controls
@variable(dop, deg2rad(-90) ≤ α ≤ deg2rad(90), DefinedOn(t))  # angle of attack (rad)
@variable(dop, deg2rad(-90) ≤ β ≤ deg2rad(1),  DefinedOn(t))  # bank angle (rad)

# Boundary conditions
@constraint(dop, initial(h) == 2.6e5);  @constraint(dop, final(h) == 0.8e5)
@constraint(dop, initial(θ) == 0)
@constraint(dop, initial(Φ) == 0)
@constraint(dop, initial(v) == 25600);  @constraint(dop, final(v) == 2500)
@constraint(dop, initial(γ) == deg2rad(-1));  @constraint(dop, final(γ) == deg2rad(-5))
@constraint(dop, initial(ψ) == deg2rad(90))

# Intermediate expressions (aerodynamics)
@expression(dop, r,     R_e + h)
@expression(dop, g,     μ / r^2)
@expression(dop, ρ,     ρ_0 * exp(-h / h_r))
@expression(dop, α_deg, (180 / pi) * α)
@expression(dop, L,     0.5 * S * ρ * v^2 * (a_0 + a_1 * α_deg))
@expression(dop, D,     0.5 * S * ρ * v^2 * (b_0 + b_1 * α_deg + b_2 * α_deg^2))

# Equations of motion
@constraint(dop, derivative(h) == v * sin(γ))
@constraint(dop, derivative(θ) == v * cos(γ) * cos(ψ) / r)
@constraint(dop, derivative(Φ) == v * cos(γ) * sin(ψ) / (r * cos(θ)))
@constraint(dop, derivative(v) == -D / m - g * sin(γ))
@constraint(dop, derivative(γ) == L * cos(β) / (m * v) + cos(γ) * (v / r - g / v))
@constraint(dop, derivative(ψ) == L * sin(β) / (m * v * cos(γ)) +
                                  v * cos(γ) * sin(ψ) * sin(θ) / (r * cos(θ)))

# Warm-start with linear guesses
JuDO.warmstart!(dop, LinearInterpolant(2.6e5,       0.8e5),        h)
JuDO.warmstart!(dop, LinearInterpolant(0.0,         deg2rad(45)),  θ)
JuDO.warmstart!(dop, LinearInterpolant(0.0,         deg2rad(50)),  Φ)
JuDO.warmstart!(dop, LinearInterpolant(25600.0,     2500.0),       v)
JuDO.warmstart!(dop, LinearInterpolant(deg2rad(-1), deg2rad(-5)),  γ)
JuDO.warmstart!(dop, LinearInterpolant(deg2rad(90), deg2rad(-20)), ψ)

# Maximise crossrange (final latitude)
@objective(dop, Max, final(θ))

JuDO.optimize!(dop)

# Extract solution trajectories
h_sol = dyn_value(dop, h)
θ_sol = dyn_value(dop, θ)
Φ_sol = dyn_value(dop, Φ)
v_sol = dyn_value(dop, v)
γ_sol = dyn_value(dop, γ)
ψ_sol = dyn_value(dop, ψ)
α_sol = dyn_value(dop, α)
β_sol = dyn_value(dop, β)

tf = phase_final(t)
ts = collect(range(0, tf; length=500))

# 3-D trajectory plot
traj = plot(
    rad2deg.(Φ_sol.(ts)),
    rad2deg.(θ_sol.(ts)),
    h_sol.(ts) ./ 1000;
    linewidth=1, legend=nothing, lc=:black,
    xlabel="Longitude (deg)", ylabel="Latitude (deg)", zlabel="Altitude (km)")
display(traj)

# Individual state and control trajectories
configs = [
    (h_sol, "Altitude (km)",           y -> y/1000),
    (θ_sol, "Latitude (deg)",          rad2deg),
    (Φ_sol, "Longitude (deg)",         rad2deg),
    (v_sol, "Velocity (km/s)",         y -> y/1000),
    (γ_sol, "Flight path angle (deg)", rad2deg),
    (ψ_sol, "Azimuth (deg)",           rad2deg),
    (α_sol, "Angle of attack (deg)",   rad2deg),
    (β_sol, "Bank angle (deg)",        rad2deg),
]
for (sol, ylabel, scale) in configs
    p = plot(ts, t -> scale(sol(t));
             legend=false, ylabel=ylabel, xlabel="Time (s)", lw=1, lc=:black)
    display(p)
end
```

**Reference:** Betts, J.T. (2010). *Practical Methods for Optimal Control and Estimation Using Nonlinear Programming*, 2nd ed. SIAM.
