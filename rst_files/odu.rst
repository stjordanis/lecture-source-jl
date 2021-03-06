.. _odu:

.. include:: /_static/includes/lecture_howto_jl.raw

.. highlight:: julia

******************************************
Job Search III: Search with Learning
******************************************

.. contents:: :depth: 2

Overview
============

In this lecture we consider an extension of the :doc:`previously studied <mccall_model>` job search model of McCall :cite:`McCall1970`

In the McCall model, an unemployed worker decides when to accept a permanent position at a specified wage, given

* his or her discount rate

* the level of unemployment compensation

* the distribution from which wage offers are drawn



In the version considered below, the wage distribution is unknown and must be learned

* The following is based on the presentation in :cite:`Ljungqvist2012`, section 6.6


Model features
----------------

* Infinite horizon dynamic programming with two states and one binary control

* Bayesian updating to learn the unknown distribution




Model
========

.. index::
    single: Models; McCall

Let's first review the basic McCall model :cite:`McCall1970` and then add the variation we want to consider



The Basic McCall Model
------------------------

.. index::
    single: McCall Model

Recall that, :doc:`in the baseline model <mccall_model>`, an unemployed worker is presented in each period with a
permanent job offer at wage :math:`W_t`

At time :math:`t`, our worker either

#. accepts the offer and works permanently at constant wage :math:`W_t`

#. rejects the offer, receives unemployment compensation :math:`c` and reconsiders next period

The wage sequence :math:`\{W_t\}` is iid and generated from known density :math:`h`

The worker aims to maximize the expected discounted sum of earnings :math:`\mathbb{E} \sum_{t=0}^{\infty} \beta^t y_t`
The function :math:`V` satisfies the recursion

.. math::
    :label: odu_odu_pv

    V(w)
    = \max \left\{
    \frac{w}{1 - \beta}, \, c + \beta \int V(w')h(w') dw'
    \right\}


The optimal policy has the form :math:`\mathbf{1}\{w \geq \bar w\}`, where
:math:`\bar w` is a constant depending called the *reservation wage*


Offer Distribution Unknown
----------------------------

Now let's extend the model by considering the variation presented in :cite:`Ljungqvist2012`, section 6.6

The model is as above, apart from the fact that

* the density :math:`h` is unknown

* the worker learns about :math:`h` by starting with a prior and updating based on wage offers that he/she observes

The worker knows there are two possible distributions :math:`F` and :math:`G` --- with densities :math:`f` and :math:`g`

At the start of time, "nature" selects :math:`h` to be either :math:`f` or
:math:`g` --- the wage distribution from which the entire sequence :math:`\{W_t\}` will be drawn

This choice is not observed by the worker, who puts prior probability :math:`\pi_0` on :math:`f` being chosen

Update rule: worker's time :math:`t` estimate of the distribution is :math:`\pi_t f + (1 - \pi_t) g`, where :math:`\pi_t` updates via

.. math::
    :label: odu_pi_rec

    \pi_{t+1}
    = \frac{\pi_t f(w_{t+1})}{\pi_t f(w_{t+1}) + (1 - \pi_t) g(w_{t+1})}


This last expression follows from Bayes' rule, which tells us that

.. math::

    \mathbb{P}\{h = f \,|\, W = w\}
    = \frac{\mathbb{P}\{W = w \,|\, h = f\}\mathbb{P}\{h = f\}}
    {\mathbb{P}\{W = w\}}
    \quad \text{and} \quad
    \mathbb{P}\{W = w\} = \sum_{\psi \in \{f, g\}} \mathbb{P}\{W = w \,|\, h = \psi\} \mathbb{P}\{h = \psi\}


The fact that :eq:`odu_pi_rec` is recursive allows us to progress to a recursive solution method


Letting

.. math::

    h_{\pi}(w) := \pi f(w) + (1 - \pi) g(w)
    \quad \text{and} \quad
    q(w, \pi) := \frac{\pi f(w)}{\pi f(w) + (1 - \pi) g(w)}


we can express the value function for the unemployed worker recursively as
follows

.. math::
    :label: odu_mvf

    V(w, \pi)
    = \max \left\{
    \frac{w}{1 - \beta}, \, c + \beta \int V(w', \pi') \, h_{\pi}(w') \, dw'
    \right\}
    \quad \text{where} \quad
    \pi' = q(w', \pi)


Notice that the current guess :math:`\pi` is a state variable, since it affects the worker's perception of probabilities for future rewards

Parameterization
------------------

Following  section 6.6 of :cite:`Ljungqvist2012`, our baseline parameterization will be

* :math:`f` is :math:`\operatorname{Beta}(1, 1)` scaled (i.e., draws are multiplied by) some factor :math:`w_m` 
  
* :math:`g` is :math:`\operatorname{Beta}(3, 1.2)` scaled (i.e., draws are multiplied by) the same factor :math:`w_m` 

* :math:`\beta = 0.95` and :math:`c = 0.6`

With :math:`w_m = 2`, the densities :math:`f` and :math:`g` have the following shape



.. code-block:: julia

  using Distributions
  using Plots

  w_max = 2
  x = linspace(0, w_max, 200)

  G = Beta(3, 1.6)
  F = Beta(1, 1)
  plot(x, pdf.(G, x/w_max)/w_max, label="g")
  plot!(x, pdf.(F, x/w_max)/w_max, label="f")
  



.. _looking_forward:

Looking Forward
------------------

What kind of optimal policy might result from :eq:`odu_mvf` and the parameterization specified above?

Intuitively, if we accept at :math:`w_a` and :math:`w_a \leq w_b`, then --- all other things being given --- we should also accept at :math:`w_b`

This suggests a policy of accepting whenever :math:`w` exceeds some threshold value :math:`\bar w`

But :math:`\bar w` should depend on :math:`\pi` --- in fact it should be decreasing in :math:`\pi` because

* :math:`f` is a less attractive offer distribution than :math:`g`
* larger :math:`\pi` means more weight on :math:`f` and less on :math:`g`

Thus larger :math:`\pi` depresses the worker's assessment of her future prospects, and relatively low current offers become more attractive

**Summary:**  We conjecture that the optimal policy is of the form
:math:`\mathbb 1\{w \geq \bar w(\pi) \}` for some decreasing function
:math:`\bar w`



Take 1: Solution by VFI
==================================

Let's set about solving the model and see how our results match with our intuition

We begin by solving via value function iteration (VFI), which is natural but ultimately turns out to be second best

The code is as follows

.. _odu_vfi_code:

.. code-block:: julia 

    #=

    @author : Spencer Lyon <spencer.lyon@nyu.edu>

    =#

    using QuantEcon
    using Interpolations

    """
    Unemployment/search problem where offer distribution is unknown

    ##### Fields

    - `β::Real` : Discount factor on (0, 1)
    - `c::Real` : Unemployment compensation
    - `F::Distribution` : Offer distribution `F`
    - `G::Distribution` : Offer distribution `G`
    - `f::Function` : The pdf of `F`
    - `g::Function` : The pdf of `G`
    - `n_w::Int` : Number of points on the grid for w
    - `w_max::Real` : Maximum wage offer
    - `w_grid::AbstractVector` : Grid of wage offers w
    - `n_π::Int` : Number of points on grid for π
    - `π_min::Real` : Minimum of π grid
    - `π_max::Real` : Maximum of π grid
    - `π_grid::AbstractVector` : Grid of probabilities π
    - `quad_nodes::Vector` : Notes for quadrature ofer offers
    - `quad_weights::Vector` : Weights for quadrature ofer offers

    """
    struct SearchProblem{TR<:Real, TI<:Integer, TF<:AbstractFloat,
                        TAVw<:AbstractVector{TF}, TAVpi<:AbstractVector{TF}}
        β::TR
        c::TR
        F::Distribution
        G::Distribution
        f::Function
        g::Function
        n_w::TI
        w_max::TR
        w_grid::TAVw
        n_π::TI
        π_min::TR
        π_max::TR
        π_grid::TAVpi
        quad_nodes::Vector{TF}
        quad_weights::Vector{TF}
    end

    """
    Constructor for `SearchProblem` with default values

    ##### Arguments

    - `β::Real(0.95)` : Discount factor in (0, 1)
    - `c::Real(0.6)` : Unemployment compensation
    - `F_a::Real(1), F_b::Real(1)` : Parameters of `Beta` distribution for `F`
    - `G_a::Real(3), G_b::Real(1.2)` : Parameters of `Beta` distribution for `G`
    - `w_max::Real(2)` : Maximum of wage offer grid
    - `w_grid_size::Integer(40)` : Number of points in wage offer grid
    - `π_grid_size::Integer(40)` : Number of points in probability grid

    ##### Notes

    There is also a version of this function that accepts keyword arguments for
    each parameter

    """

    # use key word argment
    function SearchProblem(;β::Real=0.95, c::Real=0.6, F_a::Real=1, F_b::Real=1,
                        G_a::Real=3, G_b::Real=1.2, w_max::Real=2.0,
                        w_grid_size::Integer=40, π_grid_size::Integer=40)

        F = Beta(F_a, F_b)
        G = Beta(G_a, G_b)

        # NOTE: the x./w_max)./w_max in these functions makes our dist match
        #       the scipy one with scale=w_max given
        f(x) = pdf.(F, x/w_max)/w_max
        g(x) = pdf.(G, x/w_max)/w_max

        π_min = 1e-3  # avoids instability
        π_max = 1 - π_min

        w_grid = linspace(0, w_max, w_grid_size)
        π_grid = linspace(π_min, π_max, π_grid_size)

        nodes, weights = qnwlege(21, 0.0, w_max)

        SearchProblem(β, c, F, G, f, g,
                    w_grid_size, w_max, w_grid,
                    π_grid_size, π_min, π_max, π_grid, nodes, weights)
    end

    function q(sp::SearchProblem, w, π_val)
        new_π = 1.0 / (1 + ((1 - π_val) * sp.g(w)) / (π_val * sp.f(w)))

        # Return new_π when in [π_min, π_max] and else end points
        return clamp(new_π, sp.π_min, sp.π_max)
    end

    """
    Apply the Bellman operator for a given model and initial value.

    ##### Arguments

    - `sp::SearchProblem` : Instance of `SearchProblem`
    - `v::Matrix`: Current guess for the value function
    - `out::Matrix` : Storage for output.
    - `;ret_policy::Bool(false)`: Toggles return of value or policy functions

    ##### Returns

    None, `out` is updated in place. If `ret_policy == true` out is filled with the
    policy function, otherwise the value function is stored in `out`.

    """
    function bellman_operator!(sp::SearchProblem, v::Matrix, out::Matrix;
                            ret_policy::Bool=false)
        # Simplify names
        f, g, β, c = sp.f, sp.g, sp.β, sp.c
        nodes, weights = sp.quad_nodes, sp.quad_weights

        vf = extrapolate(interpolate((sp.w_grid, sp.π_grid), v,
                        Gridded(Linear())), Flat())

        # set up quadrature nodes/weights
        # q_nodes, q_weights = qnwlege(21, 0.0, sp.w_max)

        for (w_i, w) in enumerate(sp.w_grid)
            # calculate v1
            v1 = w / (1 - β)

            for (π_j, _π) in enumerate(sp.π_grid)
                # calculate v2
                integrand(m) = [vf[m[i], q.(sp, m[i], _π)] *
                            (_π * f(m[i]) + (1 - _π) * g(m[i])) for i in 1:length(m)]
                integral = do_quad(integrand, nodes, weights)
                # integral = do_quad(integrand, q_nodes, q_weights)
                v2 = c + β * integral

                # return policy if asked for, otherwise return max of values
                out[w_i, π_j] = ret_policy ? v1 > v2 : max(v1, v2)
            end
        end
        return out
    end

    function bellman_operator(sp::SearchProblem, v::Matrix;
                            ret_policy::Bool=false)
        out_type = ret_policy ? Bool : Float64
        out = Array{out_type}(sp.n_w, sp.n_π)
        bellman_operator!(sp, v, out, ret_policy=ret_policy)
    end


    """
    Extract the greedy policy (policy function) of the model.

    ##### Arguments

    - `sp::SearchProblem` : Instance of `SearchProblem`
    - `v::Matrix`: Current guess for the value function
    - `out::Matrix` : Storage for output

    ##### Returns

    None, `out` is updated in place to hold the policy function

    """
    get_greedy!(sp::SearchProblem, v::Matrix, out::Matrix) =
        bellman_operator!(sp, v, out, ret_policy=true)

    get_greedy(sp::SearchProblem, v::Matrix) =
        bellman_operator(sp, v, ret_policy=true)

    """
    Updates the reservation wage function guess ϕ via the operator Q.

    ##### Arguments

    - `sp::SearchProblem` : Instance of `SearchProblem`
    - `ϕ::Vector`: Current guess for ϕ
    - `out::Vector` : Storage for output

    ##### Returns

    None, `out` is updated in place to hold the updated levels of ϕ
    """
    function res_wage_operator!(sp::SearchProblem, ϕ::Vector, out::Vector)
        # Simplify name
        f, g, β, c = sp.f, sp.g, sp.β, sp.c

        # Construct interpolator over π_grid, given ϕ
        ϕ_f = LinInterp(sp.π_grid, ϕ)

        # set up quadrature nodes/weights
        q_nodes, q_weights = qnwlege(7, 0.0, sp.w_max)

        for (i, _π) in enumerate(sp.π_grid)
            integrand(x) = max.(x, ϕ_f.(q.(sp, x, _π))) .* (_π * f(x) + (1 - _π) * g(x))
            integral = do_quad(integrand, q_nodes, q_weights)
            out[i] = (1 - β) * c + β * integral
        end
    end

    """
    Updates the reservation wage function guess ϕ via the operator Q.

    See the documentation for the mutating method of this function for more details
    on arguments
    """
    function res_wage_operator(sp::SearchProblem, ϕ::Vector)
        out = similar(ϕ)
        res_wage_operator!(sp, ϕ, out)
        return out
    end


The type ``SearchProblem`` is used to store parameters and methods needed to compute optimal actions

The Bellman operator is implemented as the method ``.bellman_operator()``, while ``.get_greedy()``
computes an approximate optimal policy from a guess ``v`` of the value function

We will omit a detailed discussion of the code because there is a more efficient solution method

These ideas are implemented in the ``.res_wage_operator()`` method

Before explaining it let's look at solutions computed from value function iteration

Here's the value function:



.. code-block:: julia

  using LaTeXStrings

  # Set up the problem and initial guess, solve by VFI
  sp = SearchProblem(;w_grid_size=100, π_grid_size=100)
  v_init = zeros(sp.n_w, sp.n_π) + sp.c / (1 - sp.β)
  f(x) = bellman_operator(sp, x)
  v = compute_fixed_point(f, v_init)
  policy = get_greedy(sp, v)

  # Make functions for the linear interpolants of these
  vf = extrapolate(interpolate((sp.w_grid, sp.π_grid), v, Gridded(Linear())),
                   Flat())
  pf = extrapolate(interpolate((sp.w_grid, sp.π_grid), policy,
                   Gridded(Linear())), Flat())

  function plot_value_function(;w_plot_grid_size::Integer=100,
                              π_plot_grid_size::Integer=100)
     π_plot_grid = linspace(0.001, 0.99, π_plot_grid_size)
     w_plot_grid = linspace(0, sp.w_max, w_plot_grid_size)
     Z = [vf[w_plot_grid[j], π_plot_grid[i]]
             for j in 1:w_plot_grid_size, i in 1:π_plot_grid_size]
     p = contour(π_plot_grid, w_plot_grid, Z, levels=15, alpha=0.6, 
                 fill=true, size=(400, 400), c=:lightrainbow)
     plot!(xlabel=L"$\pi$", ylabel="2", xguidefont=font(12))
     return p
  end

  plot_value_function()
  



The optimal policy:



.. code-block:: julia

  function plot_policy_function(;w_plot_grid_size::Integer=100,
                                π_plot_grid_size::Integer=100)
      π_plot_grid = linspace(0.001, 0.99, π_plot_grid_size)
      w_plot_grid = linspace(0, sp.w_max, w_plot_grid_size)
      Z = [pf[w_plot_grid[j], π_plot_grid[i]]
              for j in 1:w_plot_grid_size, i in 1:π_plot_grid_size]
      p = contour(π_plot_grid, w_plot_grid, Z, levels=1, alpha=0.6, fill=true, size=(400, 400), c=:coolwarm)
      plot!(xlabel=L"$\pi$", ylabel="wage", xguidefont=font(12), cbar=false)
      annotate!(0.4, 1.0, "reject")
      annotate!(0.7, 1.8, "accept")
      return p
  end

  plot_policy_function()
  


The code takes several minutes to run

The results fit well with our intuition from section :ref:`looking forward <looking_forward>`

* The black line in the figure above corresponds to the function :math:`\bar w(\pi)` introduced there

* It is decreasing as expected


Take 2: A More Efficient Method
==================================

Our implementation of VFI can be optimized to some degree

But instead of pursuing that, let's consider another method to solve for the optimal policy

We will use iteration with an operator that has the same contraction rate as the Bellman operator, but

* one dimensional rather than two dimensional
* no maximization step

As a consequence, the algorithm is orders of magnitude faster than VFI

This section illustrates the point that when it comes to programming, a bit of 
mathematical analysis goes a long way


Another Functional Equation
-----------------------------

To begin, note that when :math:`w = \bar w(\pi)`, the worker is indifferent
between accepting and rejecting

Hence the two choices on the right-hand side of :eq:`odu_mvf` have equal value:

.. math::
    :label: odu_mvf2

    \frac{\bar w(\pi)}{1 - \beta}
    = c + \beta \int V(w', \pi') \, h_{\pi}(w') \, dw'


Together, :eq:`odu_mvf` and :eq:`odu_mvf2` give

.. math::
    :label: odu_mvf3

    V(w, \pi) =
    \max
    \left\{
        \frac{w}{1 - \beta} ,\, \frac{\bar w(\pi)}{1 - \beta}
    \right\}


Combining :eq:`odu_mvf2` and :eq:`odu_mvf3`, we obtain

.. math::

    \frac{\bar w(\pi)}{1 - \beta}
    = c + \beta \int \max \left\{
        \frac{w'}{1 - \beta} ,\, \frac{\bar w(\pi')}{1 - \beta}
    \right\}
    \, h_{\pi}(w') \, dw'


Multiplying by :math:`1 - \beta`, substituting in :math:`\pi' = q(w', \pi)` and using :math:`\circ` for composition of functions yields

.. math::
    :label: odu_mvf4

    \bar w(\pi)
    = (1 - \beta) c +
    \beta \int \max \left\{ w', \bar w \circ q(w', \pi) \right\} \, h_{\pi}(w') \, dw'


Equation :eq:`odu_mvf4` can be understood as a functional equation, where :math:`\bar w` is the unknown function

* Let's call it the *reservation wage functional equation* (RWFE)
* The solution :math:`\bar w` to the RWFE is the object that we wish to compute


Solving the RWFE
--------------------------------

To solve the RWFE, we will first show that its solution is the
fixed point of a `contraction mapping <https://en.wikipedia.org/wiki/Contraction_mapping>`_

To this end, let

* :math:`b[0,1]` be the bounded real-valued functions on :math:`[0,1]`
* :math:`\| \psi \| := \sup_{x \in [0,1]} | \psi(x) |`

Consider the operator :math:`Q` mapping :math:`\psi \in b[0,1]` into :math:`Q\psi \in b[0,1]` via


.. math::
    :label: odu_dq

    (Q \psi)(\pi)
    = (1 - \beta) c +
    \beta \int \max \left\{ w', \psi \circ q(w', \pi) \right\} \, h_{\pi}(w') \, dw'


Comparing :eq:`odu_mvf4` and :eq:`odu_dq`, we see that the set of fixed points of :math:`Q` exactly coincides with the set of solutions to the RWFE

* If :math:`Q \bar w = \bar w` then :math:`\bar w` solves :eq:`odu_mvf4` and vice versa

Moreover, for any :math:`\psi, \phi \in b[0,1]`, basic algebra and the
triangle inequality for integrals tells us that

.. math::
    :label: odu_nt

    |(Q \psi)(\pi) - (Q \phi)(\pi)|
    \leq \beta \int
    \left|
    \max \left\{w', \psi \circ q(w', \pi) \right\}
    - \max \left\{w', \phi \circ q(w', \pi) \right\}
    \right|
    \, h_{\pi}(w') \, dw'


Working case by case, it is easy to check that for real numbers :math:`a, b, c` we always have

.. math::
    :label: odu_nt2

    | \max\{a, b\} - \max\{a, c\}| \leq | b - c|


Combining :eq:`odu_nt` and :eq:`odu_nt2` yields

.. math::
    :label: odu_nt3

    |(Q \psi)(\pi) - (Q \phi)(\pi)|
    \leq \beta \int
    \left| \psi \circ q(w', \pi) -  \phi \circ q(w', \pi) \right|
    \, h_{\pi}(w') \, dw'
    \leq \beta \| \psi - \phi \|


Taking the supremum over :math:`\pi` now gives us

.. math::
    :label: odu_rwc

    \|Q \psi - Q \phi\|
    \leq \beta \| \psi - \phi \|


In other words, :math:`Q` is a contraction of modulus :math:`\beta` on the
complete metric space :math:`(b[0,1], \| \cdot \|)`

Hence

* A unique solution :math:`\bar w` to the RWFE exists in :math:`b[0,1]`
* :math:`Q^k \psi \to \bar w` uniformly as :math:`k \to \infty`, for any :math:`\psi \in b[0,1]`

Implementation
^^^^^^^^^^^^^^^^^

These ideas are implemented in the ``.res_wage_operator()`` method from ``odu.jl`` as shown above

The method corresponds to action of the operator :math:`Q`

The following exercise asks you to exploit these facts to compute an approximation to :math:`\bar w`


Exercises
=============

.. _odu_ex1:

Exercise 1
------------

Use the default parameters and the ``.res_wage_operator()`` method to compute an optimal policy

Your result should coincide closely with the figure for the optimal policy :ref:`shown above<odu_pol_vfi>`

Try experimenting with different parameters, and confirm that the change in
the optimal policy coincides with your intuition



Solutions
==========



Exercise 1
----------

This code solves the "Offer Distribution Unknown" model by iterating on
a guess of the reservation wage function. You should find that the run
time is much shorter than that of the value function approach in
``examples/odu_vfi_plots.jl``

.. code-block:: julia

    sp = SearchProblem(π_grid_size=50)

    ϕ_init = ones(sp.n_π)
    f_ex1(x) = res_wage_operator(sp, x)
    w_bar = compute_fixed_point(f_ex1, ϕ_init)

    plot(sp.π_grid, w_bar, linewidth=2, color=:black,
    fill_between=0, fillalpha=0.15, fillcolor=:blue)
    plot!(sp.π_grid, 2*ones(length(w_bar)), linewidth=0, fill_between=w_bar,
    fillalpha=0.12, fillcolor=:green, legend=:none)
    plot!(ylims=(0, 2), annotations=[(0.42, 1.2, "reject"), (0.7, 1.8, "accept")])



The next piece of code is not one of the exercises from QuantEcon -- it's
just a fun simulation to see what the effect of a change in the
underlying distribution on the unemployment rate is

At a point in the simulation, the distribution becomes significantly
worse. It takes a while for agents to learn this, and in the meantime
they are too optimistic, and turn down too many jobs. As a result, the
unemployment rate spikes

The code takes a few minutes to run.

.. code-block:: julia

    # Set up model and compute the function w_bar
    sp = SearchProblem(π_grid_size=50, F_a=1, F_b=1)
    ϕ_init = ones(sp.n_π)
    g(x) = res_wage_operator(sp, x)
    w_bar_vals = compute_fixed_point(g, ϕ_init)
    w_bar = extrapolate(interpolate((sp.π_grid, ), w_bar_vals,
                        Gridded(Linear())), Flat())

    # Holds the employment state and beliefs of an individual agent.
    mutable struct Agent
        _π
        employed
    end

    Agent(_π=1e-3) = Agent(_π, 1)

    function update!(ag::Agent, H::Distribution)
        if ag.employed == 0
            w = rand(H) * 2   # account for scale in julia
            if w >= w_bar[ag._π]
                ag.employed = 1
            else
                ag._π = 1.0 ./ (1 .+ ((1 - ag._π) .* sp.g(w)) ./ (ag._π * sp.f(w)))
            end
        end
        nothing
    end

    num_agents = 5000
    separation_rate = 0.025  # Fraction of jobs that end in each period
    separation_num = round(Int, num_agents * separation_rate)
    agent_indices = collect(1:num_agents)
    agents = [Agent() for i=1:num_agents]
    sim_length = 600
    H = sp.G                 # Start with distribution G
    change_date = 200        # Change to F after this many periods
    unempl_rate = Vector{Float64}(sim_length)

    for i=1:sim_length
        if i % 20 == 0
            println("date = $i")
        end

        if i == change_date
            H = sp.F
        end

        # Randomly select separation_num agents and set employment status to 0
        shuffle!(agent_indices)
        separation_list = agent_indices[1:separation_num]

        for agent in agents[separation_list]
            agent.employed = 0
        end

        # update agents
        for agent in agents
            update!(agent, H)
        end
        employed = Int[agent.employed for agent in agents]
        unempl_rate[i] = 1.0 - mean(employed)
    end

    plot(unempl_rate, linewidth=2, label="unemployment rate")
    vline!([change_date], color=:red, label="")




