# Hyperopt

[![Build Status](https://travis-ci.org/baggepinnen/Hyperopt.jl.svg?branch=master)](https://travis-ci.org/baggepinnen/Hyperopt.jl)

[![Coverage Status](https://coveralls.io/repos/baggepinnen/Hyperopt.jl/badge.svg?branch=master&service=github)](https://coveralls.io/github/baggepinnen/Hyperopt.jl?branch=master)

[![codecov.io](http://codecov.io/github/baggepinnen/Hyperopt.jl/coverage.svg?branch=master)](http://codecov.io/github/baggepinnen/Hyperopt.jl?branch=master)


A package to perform hyperparameter optimization. Currently supports random search and blue-noise search.

# Usage

1. The macro `@hyperopt` takes a for-loop with an initial argument determining the number of samples to draw (`i` below).
2. The sample strategy can be specified by specifying the special keyword `sampler = Sampler(opts...)`. Available options are `RandomSampler` and `BlueNoiseSampler`.
3. The subsequent arguments to the for-loop specifies names and candidate values for different hyper parameters (`a = LinRange(1,2,1000), b = [true, false], c = exp10.(LinRange(-1,3,1000))` above). Currently uniform random sampling from the candidate values is the only supported optimizer. Log-uniform sampling is achieved with uniform sampling of a logarithmically spaced vector, e.g. `c = exp10.(LinRange(-1,3,1000))`. The parameters `i,a,b,c` can be used within the expression sent to the macro and they will hold a new value sampled from the corresponding candidate vector each iteration.

The resulting object `ho::Hyperoptimizer` holds all the sampled parameters and function values and can be queried for `minimum/maximum`, which returns the best parameters and function value found. It can also be plotted using `plot(ho)` (uses `Plots.jl`).

```julia
using Hyperopt

f(x,a,b=true;c=10) = sum(@. x + (a-3)^2 + (b ? 10 : 20) + (c-100)^2) # Function to minimize

# Main macro. The first argument to the for loop is always interpreted as the number of iterations
ho = @hyperopt for i=50,
            sampler = RandomSampler(),
            a = LinRange(1,5,1000),
            b = [true, false],
            c = exp10.(LinRange(-1,3,1000))
   print(i, "\t", a, "\t", b, "\t", c, "   \t")
   x = 100
   @show f(x,a,b,c=c)
end
1   3.910910910910911   false   0.15282140360258697     f(x, a, b, c=c) = 10090.288832348499
2   3.930930930930931   true    6.1629662551329405      f(x, a, b, c=c) = 8916.255534433481
3   2.7617617617617616  true    146.94918006248173      f(x, a, b, c=c) = 2314.282265997491
4   3.6666666666666665  false   0.3165924111983522      f(x, a, b, c=c) = 10057.226192959602
5   4.783783783783784   true    34.55719936762139       f(x, a, b, c=c) = 4395.942039196544
6   2.5895895895895897  true    4.985373463873895       f(x, a, b, c=c) = 9137.947692504491
7   1.6206206206206206  false   301.6334347259197       f(x, a, b, c=c) = 40777.94468684398
8   1.012012012012012   true    33.00034791125285       f(x, a, b, c=c) = 4602.905476253546
9   3.3583583583583585  true    193.7703337477989       f(x, a, b, c=c) = 8903.003911886599
10  4.903903903903904   true    144.26439512181574      f(x, a, b, c=c) = 2072.9615255755252
11  2.2332332332332334  false   119.97177354358843      f(x, a, b, c=c) = 519.4596697509966
12  2.369369369369369   false   117.77987011971193      f(x, a, b, c=c) = 436.52147646611473
13  3.2182182182182184  false   105.44427935261685      f(x, a, b, c=c) = 149.68779686009242
⋮

Hyperopt.Hyperoptimizer
  iterations: Int64 50
  params: Tuple{Symbol,Symbol,Symbol}
  candidates: Array{AbstractArray{T,1} where T}((3,))
  history: Array{Any}((50,))
  results: Array{Any}((50,))
  sampler: Hyperopt.RandomSampler


julia> best_params, min_f = minimum(ho)
(Real[1.62062, true, 100.694], 112.38413353985818)

julia> printmin(ho)
a = 1.62062
b = true
c = 100.694
```


The type `Hyperoptimizer` is iterable, it iterates for the specified number of iterations, each iteration providing a sample of the parameter vector, e.g.
```julia
ho = Hyperoptimizer(10, a = LinRange(1,2), b = [true, false], c = randn(100))
for (i,a,b,c) in ho
    println(i, "\t", a, "\t", b, "\t", c)
end

1   1.2244897959183674  false   0.8179751164732062
2   1.7142857142857142  true    0.6536272580487854
3   1.4285714285714286  true    -0.2737451706680355
4   1.6734693877551021  false   -0.12313108128547606
5   1.9795918367346939  false   -0.4350837079334295
6   1.0612244897959184  true    -0.2025613848798039
7   1.469387755102041   false   0.7464858339748051
8   1.8571428571428572  true    -0.9269021128132274
9   1.163265306122449   true    2.6554272337516966
10  1.4081632653061225  true    1.112896676939024
```

If used in this way, the hyperoptimizer **can not** keep track of the function values like it did when `@hyperopt` was used. To manually store the same data, consider a pattern like
```julia
ho = Hyperoptimizer(10, a = LinRange(1,2), b = [true, false], c = randn(100))
for (i,a,b,c) in ho
    res = computations(a,b,c)
    push!(ho.results, res)
    push!(ho.history, [a,b,c])
end
```

# Categorical variables
`RandomSampler` and `BlueNoiseSampler` support categorical variables which do not have a natural floating point representation, such as functions:
```julia
@hyperopt for i=20, fun = [tanh, σ, relu]
    train_network(fun)
end
```
Caveat for `BlueNoiseSampler`: see below.

# Which sampler to use?
Random is a good baseline and the default if none is chosen.

If number of iterations is smaller than, say, 200, `BlueNoiseSampler` works better than random. Caveat: `BlueNoiseSampler` needs all candidate vectors to be of equal length, i.e.,
```julia
hob = @hyperopt for i=100, sampler=BlueNoiseSampler(), a = range(1,stop=5, length=100), b = repeat([true, false],50), c = exp10.(range(-1,stop=3, length=100))
    # println(i, "\t", a, "\t", b, "\t", c)
    # print(i, " ")
    f(a,b,c=c)
end
```
where all candidate vectors are of length 100. The candidates for `b` thus had to be repeated 50 times.

`BlueNoiseSampler` performs an optimization problem in the beginning to spread out samples so as to sample the space as evenly as possible, both as measured in the full dimensional space, and in each dimension separately. Inspiration for this sampler comes from [Projective Blue-Noise Sampling](http://resources.mpi-inf.mpg.de/ProjectiveBlueNoise/ProjectiveBlueNoise.pdf) but the implemented algorithm is not the same.
The result of the blue noise optimization in 2 dimensions is visualized below. Initial values to the left and optimized to the right.
![window](bluenoise.png)

If the number of iterations is very large, the optimization problem might take long time to run in comparison to the runtime of a single experiment and random sampling will end up more effective.

# Parallel execution
The macro `@phyperopt` works in the same way as `@hyperopt` but distributes all computation on available workers. The usual caveats apply, code must be loaded on all workers etc.
