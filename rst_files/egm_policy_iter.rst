.. _egm_policy_iter:

.. include:: /_static/includes/lecture_howto_jl.raw

.. highlight:: julia

********************************************************
:index:`Optimal Growth III: The Endogenous Grid Method`
********************************************************

.. contents:: :depth: 2


Overview
============

We solved the stochastic optimal growth model using 

#. :doc:`value function iteration <optgrowth>` 
#. :doc:`Euler equation based time iteration <coleman_policy_iter>` 

We found time iteration to be significantly more accurate at each step

In this lecture we'll look at an ingenious twist on the time iteration technique called the **endogenous grid method** (EGM) 

EGM is a numerical method for implementing policy iteration invented by `Chris Carroll <http://www.econ2.jhu.edu/people/ccarroll/>`__

It is a good example of how a clever algorithm can save a massive amount of computer time

(Massive when we multiply saved CPU cycles on each implementation times the number of implementations worldwide)

The original reference is :cite:`Carroll2006`




Key Idea
==========================

Let's start by reminding ourselves of the theory and then see how the numerics fit in



Theory
------

Take the model set out in :doc:`the time iteration lecture <coleman_policy_iter>`, following the same terminology and notation 

The Euler equation is

.. math::
    :label: egm_euler

    (u'\circ c^*)(y) 
    = \beta \int (u'\circ c^*)(f(y - c^*(y)) z) f'(y - c^*(y)) z \phi(dz)


As we saw, the Coleman operator is a nonlinear operator :math:`K` engineered so that :math:`c^*` is a fixed point of :math:`K`

It takes as its argument a continuous strictly increasing consumption policy :math:`g \in \Sigma` 

It returns a new function :math:`Kg`,  where :math:`(Kg)(y)` is the :math:`c \in (0, \infty)` that solves

.. math::
    :label: egm_coledef

    u'(c) 
    = \beta \int (u' \circ g) (f(y - c) z ) f'(y - c) z \phi(dz)




Exogenous Grid
-------------------

As discussed in :doc:`the lecture on time iteration <coleman_policy_iter>`, to implement the method on a computer we need numerical approximation

In particular, we represent a policy function by a set of values on a finite grid

The function itself is reconstructed from this representation when necessary, using interpolation or some other method

:doc:`Previously <coleman_policy_iter>`, to obtain a finite representation of an updated consumption policy we

* fixed a grid of income points :math:`\{y_i\}`

* calculated the consumption value :math:`c_i` corresponding to each
  :math:`y_i` using :eq:`egm_coledef` and a root finding routine

Each :math:`c_i` is then interpreted as the value of the function :math:`K g` at :math:`y_i` 

Thus, with the points :math:`\{y_i, c_i\}` in hand, we can reconstruct :math:`Kg` via approximation 

Iteration then continues...




Endogenous Grid
--------------------

The method discussed above requires a root finding routine to find the
:math:`c_i` corresponding to a given income value :math:`y_i`

Root finding is costly because it typically involves a significant number of
function evaluations

As pointed out by Carroll :cite:`Carroll2006`, we can avoid this if
:math:`y_i` is chosen endogenously

The only assumption required is that :math:`u'` is invertible on :math:`(0, \infty)`

The idea is this:

First we fix an *exogenous* grid :math:`\{k_i\}` for capital (:math:`k = y - c`)

Then we obtain  :math:`c_i` via

.. math::
    :label: egm_getc

    c_i = 
    (u')^{-1}
    \left\{
        \beta \int (u' \circ g) (f(k_i) z ) \, f'(k_i) \, z \, \phi(dz)
    \right\}

where :math:`(u')^{-1}` is the inverse function of :math:`u'`

Finally, for each :math:`c_i` we set :math:`y_i = c_i + k_i`

It is clear that each :math:`(y_i, c_i)` pair constructed in this manner satisfies :eq:`egm_coledef`

With the points :math:`\{y_i, c_i\}` in hand, we can reconstruct :math:`Kg` via approximation as before

The name EGM comes from the fact that the grid :math:`\{y_i\}` is  determined **endogenously**


Implementation
================

Let's implement this version of the Coleman operator and see how it performs


The Operator
----------------


Here's an implementation of :math:`K` using EGM as described above

.. code-block:: julia 

    #=

    Authors: Shunsuke Hori

    =#

    """
    The approximate Coleman operator, updated using the endogenous grid
    method.

    Parameters
    ----------
    g : Function
        The current guess of the policy function
    k_grid : AbstractVector
        The set of *exogenous* grid points, for capital k = y - c
    β : AbstractFloat
        The discount factor
    u_prime : Function
        The derivative u'(c) of the utility function
    u_prime_inv : Function
        The inverse of u' (which exists by assumption)
    f : Function
        The production function f(k)
    f_prime : Function
        The derivative f'(k)
    shocks : AbstractVector
        An array of draws from the shock, for Monte Carlo integration (to
        compute expectations).
    """

    function coleman_egm(g::Function,
                        k_grid::AbstractVector,
                        β::AbstractFloat,
                        u_prime::Function,
                        u_prime_inv::Function,
                        f::Function,
                        f_prime::Function,
                        shocks::AbstractVector)

        # Allocate memory for value of consumption on endogenous grid points
        c = similar(k_grid)

        # Solve for updated consumption value
        for (i, k) in enumerate(k_grid)
            vals = u_prime.(g.(f(k) * shocks)) .* f_prime(k) .* shocks
            c[i] = u_prime_inv(β * mean(vals))
        end

        # Determine endogenous grid
        y = k_grid + c  # y_i = k_i + c_i

        # Update policy function and return
        Kg = LinInterp(y,c)
        Kg_f(x) = Kg(x)
        return Kg_f
    end


Note the lack of any root finding algorithm 

We'll also run our original implementation, which uses an exogenous grid and requires root finding, so we can perform some comparisons


.. code-block:: julia 
    :class: collapse

    #=

    Author: Shunsuke Hori

    =#

    using QuantEcon

    """
    g: input policy function
    grid: grid points
    β: discount factor
    u_prime: derivative of utility function
    f: production function
    f_prime: derivative of production function
    shocks::shock draws, used for Monte Carlo integration to compute expectation
    Kg: output value is stored
    """
    function coleman_operator!(g::AbstractVector,
                            grid::AbstractVector,
                            β::AbstractFloat,
                            u_prime::Function,
                            f::Function,
                            f_prime::Function,
                            shocks::AbstractVector,
                            Kg::AbstractVector=similar(g))

        # This function requires the container of the output value as argument Kg

        # Construct linear interpolation object #
        g_func = LinInterp(grid, g)

        # solve for updated consumption value #
        for (i, y) in enumerate(grid)
            function h(c)
                vals = u_prime.(g_func.(f(y - c) * shocks)) .* f_prime(y - c) .* shocks
                return u_prime(c) - β * mean(vals)
            end
            Kg[i] = brent(h, 1e-10, y - 1e-10)
        end
        return Kg
    end

    # The following function does NOT require the container of the output value as argument
    function coleman_operator(g::AbstractVector,
                            grid::AbstractVector,
                            β::AbstractFloat,
                            u_prime::Function,
                            f::Function,
                            f_prime::Function,
                            shocks::AbstractVector)

        return coleman_operator!(g, grid, β, u_prime,
                                f, f_prime, shocks, similar(g))
    end

Let's test out the code above on some example parameterizations, after the following imports



.. code-block:: julia

    using PyPlot
    using LaTeXStrings





Testing on the Log / Cobb--Douglas case
------------------------------------------


As we :doc:`did for value function iteration <optgrowth>` and :doc:`time iteration <coleman_policy_iter>`, let's start by testing our method with the log-linear benchmark




The first step is to bring in the model that we used in the :doc:`Coleman policy function iteration <coleman_policy_iter>`

.. code-block:: julia 

    #=

    Author: Shunsuke Hori

    =#

    struct Model{TF <: AbstractFloat, TR <: Real, TI <: Integer}
        α::TR              # Productivity parameter
        β::TF              # Discount factor
        γ::TR              # risk aversion
        μ::TR              # First parameter in lognorm(μ, σ)
        s::TR              # Second parameter in lognorm(μ, σ)
        grid_min::TR       # Smallest grid point
        grid_max::TR       # Largest grid point
        grid_size::TI      # Number of grid points
        u::Function        # utility function
        u_prime::Function  # derivative of utility function
        f::Function        # production function
        f_prime::Function  # derivative of production function
        grid::Vector{TR}   # grid
    end

    """
    construct Model instance using the information of parameters and functional form

    arguments: see above

    return: Model type instance
    """
    function Model(; α::Real=0.65,                      # Productivity parameter
                    β::AbstractFloat=0.95,             # Discount factor
                    γ::Real=1.0,                       # risk aversion
                    μ::Real=0.0,                       # First parameter in lognorm(μ, σ)
                    s::Real=0.1,                       # Second parameter in lognorm(μ, σ)
                    grid_min::Real=1e-6,               # Smallest grid point
                    grid_max::Real=4.0,                # Largest grid point
                    grid_size::Integer=200,            # Number of grid points
                    u::Function= c->(c^(1-γ)-1)/(1-γ), # utility function
                    u_prime::Function = c-> c^(-γ),    # u'
                    f::Function = k-> k^α,             # production function
                    f_prime::Function = k -> α*k^(α-1) # f'
                    )

        grid = collect(linspace(grid_min, grid_max, grid_size))

        if γ == 1                                       # when γ==1, log utility is assigned
            u_log(c) = log(c)
            m = Model(α, β, γ, μ, s, grid_min, grid_max,
                    grid_size, u_log, u_prime, f, f_prime, grid)
        else
            m = Model(α, β, γ, μ, s, grid_min, grid_max,
                    grid_size, u, u_prime, f, f_prime, grid)
        end
        return m
    end




Next we generate an instance




.. code-block:: julia

    mlog = Model(γ=1.0) # Log Linear model






We also need some shock draws for Monte Carlo integration

.. code-block:: julia

    shock_size = 250     # Number of shock draws in Monte Carlo integral
    shocks = exp.(mlog.μ + mlog.s * randn(shock_size))




As a preliminary test, let's see if :math:`K c^* = c^*`, as implied by the theory



.. code-block:: julia

    c_star(y) = (1 - mlog.α * mlog.β) * y

    # == Some useful constants == #
    ab = mlog.α * mlog.β
    c1 = log(1 - ab) / (1 - mlog.β)
    c2 = (mlog.μ + mlog.α * log(ab)) / (1 - mlog.α)
    c3 = 1 / (1 - mlog.β)
    c4 = 1 / (1 - ab)

    v_star(y) = c1 + c2 * (c3 - c4) + c4 * log(y)


.. code-block:: julia

    function verify_true_policy(m::Model,
                                shocks::AbstractVector,
                                c_star::Function)
	k_grid = m.grid
        c_star_new = coleman_egm(c_star, k_grid, m.β, m.u_prime,
                                 m.u_prime, m.f, m.f_prime, shocks)
        fig, ax = subplots(figsize=(9, 6))

        ax[:plot](k_grid, c_star.(k_grid), label=L"optimal policy $c^*$")
        ax[:plot](k_grid, c_star_new.(k_grid), label=L"$Kc^*$")
        ax[:legend](loc="upper left")
    end

.. code-block:: julia

    verify_true_policy(mlog,shocks,c_star)



Notice that we're passing `u_prime` to `coleman_egm` twice

The reason is that, in the case of log utility, :math:`u'(c) = (u')^{-1}(c) = 1/c`

Hence `u_prime` and `u_prime_inv` are the same

We can't really distinguish the two plots

In fact it's easy to see that the difference is essentially zero:



.. code-block:: julia

    c_star_new = coleman_egm(c_star, mlog.grid, mlog.β, mlog.u_prime,
                             mlog.u_prime, mlog.f, mlog.f_prime, shocks)
    maximum(abs, (c_star_new.(mlog.grid) - c_star.(mlog.grid)))






Next let's try iterating from an arbitrary initial condition and see if we
converge towards :math:`c^*`


Let's start from the consumption policy that eats the whole pie: :math:`c(y) = y`



.. code-block:: julia

    g_init(x) = x
    n = 15
    function check_convergence(m::Model,
                               shocks::AbstractVector,
                               c_star::Function,
                               g_init::Function,
                               n_iter::Integer)
	k_grid = m.grid
        g = g_init
        fig, ax = subplots(figsize = (9, 6))
        jet = ColorMap("jet")
        plot(m.grid, g.(m.grid),
             color=jet(0), lw=2, alpha=0.6, label=L"initial condition $c(y) = y$")
        for i in 1:n_iter
            new_g = coleman_egm(g, k_grid,
                                m.β, m.u_prime, m.u_prime, m.f, m.f_prime, shocks)
            g = new_g
            ax[:plot](k_grid,new_g.(k_grid), alpha=0.6, color=jet(i / n_iter), lw=2)
        end

        ax[:plot](k_grid, c_star.(k_grid),
                  "k-", lw=2, alpha=0.8, label= L"true policy function $c^*$")
        ax[:legend](loc="upper left")
    end

.. code-block:: julia

    check_convergence(mlog, shocks, c_star, g_init, n)



We see that the policy has converged nicely, in only a few steps


Speed
========

Now let's compare the clock times per iteration for the standard Coleman
operator (with exogenous grid) and the EGM version

We'll do so using the CRRA model adopted in the exercises of the :doc:`Euler equation time iteration lecture <coleman_policy_iter>`

Here's the model and some convenient functions



.. code-block:: julia

    mcrra = Model(α = 0.65, β = 0.95, γ = 1.5)
    u_prime_inv(c) = c^(-1/mcrra.γ)




Here's the result



.. code-block:: julia

    function compare_clock_time(m::Model,
                                u_prime_inv::Function;
                                sim_length::Integer=20)
        k_grid = m.grid
        crra_coleman(g) = coleman_operator(g, k_grid, m.β, m.u_prime,
                                           m.f, m.f_prime, shocks)

        crra_coleman_egm(g) = coleman_egm(g, k_grid, m.β, m.u_prime, 
                                          u_prime_inv, m.f, m.f_prime, shocks)

        print("Timing standard Coleman policy function iteration")
        g_init = k_grid
        g = g_init
        @time begin
            for i in 1:sim_length
                new_g = crra_coleman(g)
                g = new_g
            end
        end

        print("Timing policy function iteration with endogenous grid")
        g_init_egm(x) = x
        g = g_init_egm
        @time begin
            for i in 1:sim_length
                new_g = crra_coleman_egm(g)
                g = new_g
            end
        end
        return nothing
    end
    compare_clock_time(mcrra, u_prime_inv, sim_length=20)


We see that the EGM version is more than 9 times faster



At the same time, the absence of numerical root finding means that it is
typically more accurate at each step as well