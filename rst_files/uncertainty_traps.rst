.. _uncertainty_traps:

.. include:: /_static/includes/lecture_howto_jl.raw

***************************************
Uncertainty Traps
***************************************

.. highlight:: julia


Overview
============

In this lecture we study a simplified version of an uncertainty traps model of Fajgelbaum, Schaal and Taschereau-Dumouchel :cite:`fun`

The model features self-reinforcing uncertainty that has big impacts on economic activity

In the model, 

* Fundamentals  vary stochastically and are not fully observable

* At any moment there are both active and inactive entrepreneurs; only active entrepreneurs produce

* Agents -- active and inactive entrepreuneurs --  have beliefs about the fundamentals expressed as probability distributions

* Greater uncertainty means greater dispersions of these distributions

* Entrepreneurs are risk averse and hence less inclined to be active  when uncertainty is high

* The output of active entrepreneurs is observable, supplying a noisy signal that helps everyone inside the model infer fundamentals

* Entrepreneurs update their beliefs about fundamentals using Bayes' Law, implemented via :doc:`Kalman filtering <kalman>` 


Uncertainty traps emerge because:

* High uncertainty discourages entrepreneurs from becoming active

* A low level of participation -- i.e., a smaller number of active entrepreneurs -- diminishes the flow of information about fundamentals

* Less information translates to higher uncertainty, further discouraging entrepreneurs from choosing to be active, and so on

Uncertainty traps stem from a positive externality: high aggregate economic activity levels generates valuable information





The Model
===============


The original model described in :cite:`fun` has many interesting moving parts 

Here we examine a simplified version that nonetheless captures many of the key ideas



Fundamentals
--------------

The evolution of the fundamental process :math:`\{\theta_t\}` is given by

.. math::

    \theta_{t+1} = \rho \theta_t + \sigma_{\theta} w_{t+1}


where 

* :math:`\sigma_\theta > 0` and :math:`0 < \rho < 1`

* :math:`\{w_t\}` is IID and standard normal

The random variable :math:`\theta_t` is not observable at any time



Output
-----------

There is a total :math:`\bar M` of risk averse entrepreneurs

Output of the :math:`m`-th entrepreneur, conditional on being active in the market at
time :math:`t`, is equal to 

.. math::
    :label: xgt

    x_m = \theta + \epsilon_m 
    \quad \text{where} \quad
    \epsilon_m \sim N \left(0, \gamma_x^{-1} \right)


Here the time subscript has been dropped to simplify notation

The inverse of the shock variance, :math:`\gamma_x`, is called the shock's **precision**

The higher is the precision, the more informative :math:`x_m` is about the fundamental

Output shocks are independent across time and firms





Information and Beliefs
----------------------------

All entrepreneurs start with identical beliefs about :math:`\theta_0`

Signals are publicly observable and hence all agents have identical beliefs always

Dropping time subscripts, beliefs for current :math:`\theta` are represented by the normal
distribution :math:`N(\mu, \gamma^{-1})`

Here :math:`\gamma` is the precision of beliefs; its inverse is the degree of uncertainty

These parameters are updated by Kalman filtering 

Let

* :math:`\mathbb M \subset \{1, \ldots, \bar M\}` denote the set of currently active firms 

* :math:`M := |\mathbb M|` denote the number of currently active firms 

* :math:`X` be the average output :math:`\frac{1}{M} \sum_{m \in \mathbb M} x_m` of the active firms

With this notation and primes for next period values, we can write the updating of the mean and precision via


.. math::
    :label: update_mean

    \mu' = \rho \frac{\gamma \mu + M \gamma_x X}{\gamma + M \gamma_x}


.. math::
    :label: update_prec

    \gamma' = 
        \left(
        \frac{\rho^2}{\gamma + M \gamma_x} + \sigma_\theta^2
        \right)^{-1}


These are standard Kalman filtering results applied to the current setting

Exercise 1 provides more details on how :eq:`update_mean` and :eq:`update_prec` are derived, and then asks you to fill in remaining steps

The next figure plots the law of motion for the precision in :eq:`update_prec`
as a 45 degree diagram, with one curve for each :math:`M \in \{0, \ldots, 6\}`

The other parameter values are :math:`\rho = 0.99, \gamma_x = 0.5, \sigma_\theta =0.5`

.. figure:: /_static/figures/uncertainty_traps_45.png
   :scale: 100%


Points where the curves hit the 45 degree lines are  long run steady
states for precision for different values of :math:`M`

Thus, if one of these values for :math:`M` remains fixed, a corresponding steady state is the equilibrium level of precision

* high values of :math:`M` correspond to greater information about the
  fundamental, and hence more precision in steady state

* low values of :math:`M` correspond to less information and more uncertainty in steady state

In practice, as we'll see, the number of active firms fluctuates stochastically


Participation
----------------

Omitting time subscripts once more, entrepreneurs enter the market in the current period if

.. math::
    :label: pref1

    \mathbb E [ u(x_m - F_m) ] > c


Here

* the mathematical expectation of :math:`x_m` is based on :eq:`xgt` and beliefs :math:`N(\mu, \gamma^{-1})` for :math:`\theta`

* :math:`F_m` is a stochastic but previsible fixed cost, independent across time and firms

* :math:`c` is a constant reflecting opportunity costs

The statement that :math:`F_m` is previsible means that it is realized at the start of the period and treated as a constant in :eq:`pref1`



The utility function has the constant absolute risk aversion form

.. math::
    :label: pref2

    u(x) = \frac{1}{a} \left(1 - \exp(-a x) \right)


where :math:`a` is a positive parameter

Combining :eq:`pref1` and :eq:`pref2`, entrepreneur :math:`m` participates in the market (or is said to be active) when


.. math::

    \frac{1}{a} 
        \left\{
            1 - \mathbb E [ \exp \left(
                -a (\theta + \epsilon_m -  F_m)
                    \right) ]
        \right\}
            > c


Using standard formulas for expectations of `lognormal <https://en.wikipedia.org/wiki/Log-normal_distribution>`_ random variables, this is equivalent to the condition

.. math::
    :label: firm_test

    \psi(\mu, \gamma, F_m)
    := 
    \frac{1}{a} 
        \left(
            1 - \exp \left(
                -a \mu + a F_m
                + \frac{a^2 \left( \frac{1}{\gamma} + \frac{1}{\gamma_x} \right)}{2}
                    \right) 
        \right)
            - c
            > 0


Implementation
===============


We want to simulate this economy

As a first step, let's put together a type that bundles 

* the parameters, the current value of :math:`\theta` and the current values of the
  two belief parameters :math:`\mu` and :math:`\gamma` 

* methods to update :math:`\theta`, :math:`\mu` and :math:`\gamma`, as well as to determine the number of active firms and their outputs

The updating methods follow the laws of motion for :math:`\theta`, :math:`\mu` and :math:`\gamma` given above

The method to evaluate the number of active firms generates :math:`F_1,
\ldots, F_{\bar M}` and tests condition :eq:`firm_test` for each firm

The function `UncertaintyTrapEcon` encodes as default values the parameters we'll use in the simulations below


.. code-block:: julia 

    mutable struct UncertaintyTrapEcon{TF<:AbstractFloat, TI<:Integer}
        a::TF          # Risk aversion
        γ_x::TF        # Production shock precision
        ρ::TF          # Correlation coefficient for θ
        σ_θ::TF        # Standard dev of θ shock
        num_firms::TI  # Number of firms
        σ_F::TF        # Std dev of fixed costs
        c::TF          # External opportunity cost
        μ::TF          # Initial value for μ
        γ::TF          # Initial value for γ
        θ::TF          # Initial value for θ
        σ_x::TF        # Standard deviation of shock
    end

    function UncertaintyTrapEcon(;a::AbstractFloat=1.5, γ_x::AbstractFloat=0.5,
                                ρ::AbstractFloat=0.99, σ_θ::AbstractFloat=0.5,
                                num_firms::Integer=100, σ_F::AbstractFloat=1.5,
                                c::AbstractFloat=-420.0, μ_init::AbstractFloat=0.0,
                                γ_init::AbstractFloat=4.0,
                                θ_init::AbstractFloat=0.0)
        σ_x = sqrt(a / γ_x)
        UncertaintyTrapEcon(a, γ_x, ρ, σ_θ, num_firms, σ_F, c, μ_init,
                            γ_init, θ_init, σ_x)

    end

    function ψ(uc::UncertaintyTrapEcon, F::Real)
        temp1 = -uc.a * (uc.μ - F)
        temp2 = 0.5 * uc.a^2 * (1 / uc.γ + 1 / uc.γ_x)
        return (1 / uc.a) * (1 - exp(temp1 + temp2)) - uc.c
    end

    function update_beliefs!(uc::UncertaintyTrapEcon, X::Real, M::Real)
        # Simplify names
        γ_x, ρ, σ_θ = uc.γ_x, uc.ρ, uc.σ_θ

        # Update μ
        temp1 = ρ * (uc.γ * uc.μ + M * γ_x * X)
        temp2 = uc.γ + M * γ_x
        uc.μ =  temp1 / temp2

        # Update γ
        uc.γ = 1 / (ρ^2 / (uc.γ + M * γ_x) + σ_θ^2)
    end

    update_θ!(uc::UncertaintyTrapEcon, w::Real) =
        (uc.θ = uc.ρ * uc.θ + uc.σ_θ * w)

    function gen_aggregates(uc::UncertaintyTrapEcon)
        F_vals = uc.σ_F * randn(uc.num_firms)

        M = sum(ψ.(Ref(uc), F_vals) .> 0)  # Counts number of active firms
        if M > 0
            x_vals = uc.θ .+ uc.σ_x * randn(M)
            X = mean(x_vals)
        else
            X = 0.0
        end
        return X, M
    end


In the results below we use this code to simulate time series for the major variables


Results
===============

Let's look first at the dynamics of :math:`\mu`, which the agents use to track :math:`\theta`

.. figure:: /_static/figures/uncertainty_traps_mu.png
   :scale: 100%

We see that :math:`\mu` tracks :math:`\theta` well when there are sufficient firms in the market

However, there are times when :math:`\mu` tracks :math:`\theta` poorly due to
insufficient information

These are episodes where the uncertainty traps take hold

During these episodes

* precision is low and uncertainty is high

* few firms are in the market


To get a clearer idea of the dynamics, let's look at all the main time series
at once, for a given set of shocks

.. figure:: /_static/figures/uncertainty_traps_sim.png
   :scale: 100%


Notice how the traps only take hold after a sequence of bad draws for the fundamental

Thus, the model gives us a *propagation mechanism* that maps bad random draws into long downturns in economic activity



Exercises
==============

.. _uncertainty_traps_ex1:


Exercise 1
------------

Fill in the details behind :eq:`update_mean` and :eq:`update_prec` based on
the following standard result (see, e.g., p. 24 of :cite:`young2005`)

**Fact** Let :math:`\mathbf x = (x_1, \ldots, x_M)` be a vector of IID draws
from common distribution :math:`N(\theta, 1/\gamma_x)`
and let :math:`\bar x` be the sample mean.  If :math:`\gamma_x`
is known and the prior for :math:`\theta` is :math:`N(\mu, 1/\gamma)`, then the posterior
distribution of :math:`\theta` given :math:`\mathbf x` is 

.. math::

    \pi(\theta \,|\, \mathbf x) = N(\mu_0, 1/\gamma_0)


where

.. math::

    \mu_0 = \frac{\mu \gamma + M \bar x \gamma_x}{\gamma + M \gamma_x}
    \quad \text{and} \quad
    \gamma_0 = \gamma + M \gamma_x


Exercise 2
------------

Modulo randomness, replicate the simulation figures shown above

* Use the parameter values listed as defaults in the function `UncertaintyTrapEcon`





Solutions
=========

Exercise 1
----------

This exercise asked you to validate the laws of motion for
:math:`\gamma` and :math:`\mu` given in the lecture, based on the stated
result about Bayesian updating in a scalar Gaussian setting

The stated result tells us that after observing average output :math:`X` of the
:math:`M` firms, our posterior beliefs will be

.. math::


       N(\mu_0, 1/\gamma_0)

where

.. math::


       \mu_0 = \frac{\mu \gamma + M X \gamma_x}{\gamma + M \gamma_x}
       \quad \text{and} \quad
       \gamma_0 = \gamma + M \gamma_x

If we take a random variable :math:`\theta` with this distribution and
then evaluate the distribution of :math:`\rho \theta + \sigma_\theta w`
where :math:`w` is independent and standard normal, we get the
expressions for :math:`\mu'` and :math:`\gamma'` given in the lecture.

Exercise 2
==========

First let's replicate the plot that illustrates the law of motion for
precision, which is

.. math::


       \gamma_{t+1} =
           \left(
           \frac{\rho^2}{\gamma_t + M \gamma_x} + \sigma_\theta^2
           \right)^{-1}

Here :math:`M` is the number of active firms. The next figure plots
:math:`\gamma_{t+1}` against :math:`\gamma_t` on a 45 degree diagram for
different values of :math:`M`

.. code-block:: julia

    using QuantEcon, Gadfly, DataFrames, LaTeXStrings

.. code-block:: julia

    econ = UncertaintyTrapEcon()
    ρ, σ_θ, γ_x = econ.ρ, econ.σ_θ, econ.γ_x # simplify names

    # grid for γ and γ_{t+1}
    γ = range(1e-10, stop = 3, length = 200)
    M_range = 0:6
    γp = 1 ./ (ρ^2 ./ (γ .+ γ_x .* M_range') .+ σ_θ^2)

    p1 = plot(x=repeat(collect(γ), outer=[length(M_range)+1]),
         y=vec([γ γp]),
         color=repeat(["45 Degree"; map(string, M_range)], inner=[length(γ)]),
         Geom.line, Guide.colorkey(title="M"), Guide.xlabel("γ"), Guide.ylabel("γ'"))


The points where the curves hit the 45 degree lines are the long run
steady states corresponding to each :math:`M`, if that value of
:math:`M` was to remain fixed. As the number of firms falls, so does the
long run steady state of precision.

Next let's generate time series for beliefs and the aggregates -- that
is, the number of active firms and average output.

.. code-block:: julia

    function QuantEcon.simulate(uc::UncertaintyTrapEcon{TF, TI}, capT::TI=2000) where {TF <: AbstractFloat, TI <: Integer}

        # allocate memory
        μ_vec = Vector{TF}(undef, capT)
        θ_vec = Vector{TF}(undef, capT)
        γ_vec = Vector{TF}(undef, capT)
        X_vec = Vector{TF}(undef, capT)
        M_vec = Vector{TI}(undef, capT)



        # set initial using fields from object
        μ_vec[1] = uc.μ
        γ_vec[1] = uc.γ
        θ_vec[1] = 0

        # draw standard normal shocks
        w_shocks = randn(capT)

        for t=1:capT-1
            X, M = gen_aggregates(uc)
            X_vec[t] = X
            M_vec[t] = M

            update_beliefs!(uc, X, M)
            update_θ!(uc, w_shocks[t])

            μ_vec[t+1] = uc.μ
            γ_vec[t+1] = uc.γ
            θ_vec[t+1] = uc.θ
        end

        # Record final values of aggregates
        X, M = gen_aggregates(uc)
        X_vec[end] = X
        M_vec[end] = M

        return μ_vec, γ_vec, θ_vec, X_vec, M_vec
    end

First let's see how well :math:`\mu` tracks :math:`\theta` in these
simulations

.. code-block:: julia

    using Random 
    Random.seed!(42)  # set random seed for reproducible results
    μ_vec, γ_vec, θ_vec, X_vec, M_vec = simulate(econ)

    p2 = plot(x=repeat(collect(1:length(μ_vec)), outer=[2]),
         y=[μ_vec; θ_vec],
         color=repeat(["μ", "θ"], inner=[length(μ_vec)]),
         Geom.line, Guide.colorkey(title="Variable"))



Now let's plot the whole thing together

.. code-block:: julia

    mdf = DataFrame(t=1:length(θ_vec), θ=θ_vec, μ=μ_vec, γ=γ_vec, M=M_vec)
    p3 = plot(stack(mdf, collect(2:5)), x="t",
         ygroup="variable",
         y="value",
         Geom.subplot_grid(Geom.line, free_y_axis=true))




