@def title = "About"
@def hascode = true
@def tags = ["about"]

# About JuDO

For an overview of the project motivation and design philosophy, see the [home page](/).

---

## The packages

| Package | Role | Status |
|---------|------|--------|
| [JuDO.jl](https://github.com/JuDO-dev/JuDO.jl) | User-facing modelling language | Active development |
| [DynOptInterface.jl](https://github.com/JuDO-dev/DynOptInterface.jl) | Mathematical abstraction layer (DOI) | v0.3.0 |
| [Interesso.jl](https://github.com/JuDO-dev/Interesso.jl) | Integrated Residuals solver via Ipopt | Active development |

See the [Packages](/Packages/) page for full descriptions, API references, and installation instructions.

---

## Contributors

**Team owner:** Professor Eric Kerrigan ([@erickerrigan](https://github.com/erickerrigan))

**JuDO:** Haochen Tao ([@shawn-tao01](https://github.com/shawn-tao01)), Eduardo Vila ([@e-duar-do](https://github.com/e-duar-do))

**DynOptInterface:** Haochen Tao ([@shawn-tao01](https://github.com/shawn-tao01)), Eduardo Vila ([@e-duar-do](https://github.com/e-duar-do))

**Interesso:** Lucian Nita ([@LucianNita](https://github.com/LucianNita)), Eduardo Vila ([@e-duar-do](https://github.com/e-duar-do)), Lester ([@Kailai-Shi](https://github.com/Kailai-Shi)), Haochen Tao ([@shawn-tao01](https://github.com/shawn-tao01))

---

## Organisation

JuDO is developed and maintained by the [JuDO-dev](https://github.com/JuDO-dev) organisation on GitHub.

---

## Contributing

All packages are open to contributions. The best starting points are:

- **Bug reports and feature requests**: open an issue on the relevant GitHub repository
- **New solvers**: implement your solver with a wrapper compatible to the [DynOptInterface](https://github.com/JuDO-dev/DynOptInterface.jl) to make your solver accessible from JuDO
- **Documentation**: improvements to any of the package docs are always welcome