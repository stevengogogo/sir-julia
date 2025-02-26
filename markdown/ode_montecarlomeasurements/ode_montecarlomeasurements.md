# Uncertainty propagation applied to ordinary differential equation model using MonteCarloMeasurements.jl
Simon Frost (@sdwfrost), 2022-02-14

## Introduction

The classical ODE version of the SIR model is:

- Deterministic
- Continuous in time
- Continuous in state

One elegant approach to investigating how uncertainty in parameters propagates to the output involves the use of `MonteCarloMeasurements.jl`. Parameter values and initial conditions can be represented by a set of `Particle`s, which can accommodate arbitrary patterns of variation. These can be included in `Diffe
` solvers in the same way as other, simpler types such as `Float64`. This approach is well suited to non-linear patterns of uncertainty propagation.

## Libraries

```julia
using DifferentialEquations
using OrdinaryDiffEq
using Distributions
using MonteCarloMeasurements
using StatsBase
using Plots
```




## Transitions

The following function provides the derivatives of the model, which it changes in-place. State variables and parameters are unpacked from `u` and `p`.

```julia
function sir_ode!(du,u,p,t)
    (S,I,R) = u
    (β,c,γ) = p
    N = S+I+R
    @inbounds begin
        du[1] = -β*c*I/N*S
        du[2] = β*c*I/N*S - γ*I
        du[3] = γ*I
    end
    nothing
end;
```




## Time domain

We set the timespan for simulations, `tspan`, initial conditions, `u0`, and parameter values, `p`. We will set the maximum time to be high, as we will be using a callback in order to stop the integration early.

```julia
δt = 1.0
tmax = 40.0
tspan = (0.0,tmax);
```





## Initial conditions and parameter values

We first set fixed parameters, in this case, the total population size, `N`.

```julia
N = 1000.0;
```




We then generate a random sample of parameter values as well as the initial number of infected individuals. Rather than a full factorial design, we use `LatinHypercubeSample` from the `QuasiMonteCarlo.jl` package. We specify lower (`lb`) and upper (`ub`) bounds for each parameter.

```julia
n_samples = 1000; # Number of samples
```


```julia
p = [Particles(n_samples,Uniform(0.01,0.1)),
      Particles(n_samples,Uniform(5,20.0)),
      Particles(n_samples,Uniform(0.1,1.0))]
```

```
3-element Vector{MonteCarloMeasurements.Particles{Float64, 1000}}:
  0.055 ± 0.026
 12.5 ± 4.3
  0.55 ± 0.26
```



```julia
I₀=Particles(n_samples,Uniform(1.0,50.0))
u0 = [N-I₀,I₀,0.0]
```

```
3-element Vector{MonteCarloMeasurements.Particles{Float64, 1000}}:
 974.0 ± 14.0
  25.5 ± 14.0
   0.0
```





## Running the model

```julia
prob_ode = ODEProblem(sir_ode!,u0,tspan,p);
```


```julia
sol_ode = solve(prob_ode, Tsit5(), dt=δt);
```




## Post-processing

Here are the (uncertain) states at time `t=20.0`.

```julia
s20 = sol_ode(20.0)
```

```
3-element Vector{MonteCarloMeasurements.Particles{Float64, 1000}}:
 588.0 ± 370.0
  28.0 ± 54.0
 384.0 ± 350.0
```





These states are markedly different from Gaussian distributions, as the parameter set has combinations that lead to outbreaks (R₀>1) as well as to fade-out.

```julia
l = @layout [a b c]
binwidth = 50
pl1 = histogram(s20[1],bins=0:binwidth:N, title="S(20)", xlabel="S", ylabel="Frequency", color=:blue)
pl2 = histogram(s20[2],bins=0:binwidth:N, title="I(20)", xlabel="I", ylabel="Frequency", color=:red)
pl3 = histogram(s20[3],bins=0:binwidth:N, title="R(20)", xlabel="R", ylabel="Frequency", color=:green)
plot(pl1,pl2,pl3,layout=l,legend=false)
```

![](figures/ode_montecarlomeasurements_11_1.png)



Further processing of the output can be performed after converting to `Array`s, e.g. Kendalls rank correlation of β, c, γ, and I₀ against `S(20)`, `I(20)`, and `R(20)`.

```julia
corkendall(hcat(Array(p),Array(I₀)),Array(s20))
```

```
4×3 Matrix{Float64}:
 -0.432496   0.207059    0.455307
 -0.272136   0.0889489   0.291528
  0.387788  -0.607596   -0.323688
 -0.124452  -0.0522002   0.143972
```


