# Automated Search for Optimal Parallelization Strategies in CATFlow

This document outlines the design and challenges of creating an automated search algorithm for finding optimal parallelization plans within the CATFlow framework. It organizes initial brainstorming into a structured approach, addressing key problems and proposing potential solutions.

## 1. Problem Definition: The Search for an Optimal Plan

The core challenge is to find an optimal, fully-instantiated parallelization plan from the vast strategy space defined by CATFlow's data-monistic abstractions. A parallelization plan is a complete specification of the space-time trajectory for every tensor in the computation graph.

- **Input**: A neural network model (dataflow graph) and a set of hardware resources.
- **Search Space**: The set of all valid parallelization plans constructible using CATFlow's primitives, respecting an initial set of high-level conditions (e.g., memory capacity per device).
- **Objective**: Find a plan that minimizes a cost function, typically the estimated end-to-end training time (makespan), which includes both computation and communication costs.
- **Complexity**: This is an NP-hard combinatorial optimization problem. Therefore, we must rely on heuristics and approximation algorithms to find a high-quality solution in a reasonable amount of time.

**Characteristics of the Search Space:**  
The search space is entirely defined by CATs (Constraint-aware Tensors), where a "constraint" refers specifically to the relationship between one or more tensors and their space-time coordinates. For example, a constraint might specify that Tensor1 must reside on Device1 at a particular moment, or that Tensor1 and Tensor2 must meet on Device2 at the same time (enabling a computation that produces Tensor3). This approach describes the strategy space from a perspective fundamentally different from the traditional operator-centric view. In unconstrained cases—such as when the device placement of Tensor1 is unspecified—the strategy space naturally includes all possible placements and corresponding strategies for that tensor. Throughout the search process, we aim to preserve this "purity", relying solely on CAT-related concepts (Tensor, event, constraint) to define and explore the space. Importantly, the constraints discussed here do not include resource limitations like memory capacity; they refer exclusively to the mapping between tensors and their space-time trajectories.

It can be argued that programs constructed using primitives in CATFlow are a form of refinement  constraint programming model. Rather than specifying an explicit sequence of steps to execute, these programs define the properties and constraints that any valid solution (i.e., parallelization plan) must satisfy. In this paradigm, the focus shifts from prescribing how to achieve a result to describing what constitutes an acceptable result. The search algorithm is then responsible for exploring the space of all possible solutions that meet these constraints, imposing new constraints, and ultimately selecting one that optimizes the desired objective.

## 2. High-Level Search Paradigms

Two primary paradigms for this search problem are considered: dynamic runtime scheduling and static global optimization.

### 2.1. Dynamic Runtime Scheduling

This approach involves making scheduling decisions for each CAT incrementally as time progresses, based on the current system state or "environmental pressure" (e.g., device memory usage, network congestion).

- **Concept**: A greedy, event-driven scheduler that decides the "next step" for a tensor (e.g., move to idle memory, begin computation) when a specific event occurs.
- **Defining the Heuristic**: A key challenge is defining a heuristic function that accurately
    - **Local Optima**: Greedy decisions are notoriously prone to leading to globally suboptimal solutions. An action that seems optimal now (e.g., offloading a tensor to free up memory) might lead to severe data-fetching stalls later.
    - **Infeasibility**: This approach cannot guarantee that future constraints will be met. The CATFlow primitives can define constraints across different moments in time (e.g., two tensors must meet at time `T`). A greedy choice at time `t < T` could make it isn't sensitive to satisfy the future constraint at `T`.
    - **Oscillation & Instability**: The system could enter a state where it repeatedly moves data back and forth without making forward progress, especially if the "pressure" metrics are not designed carefully.
    - **High Overhead**: Making fine-grained scheduling decisions at runtime for thousands of tensors would introduce significant computational overhead, potentially slowing down the training process itself.
    - **Step Selection Challenge**: Determining which scheduling action will yield the greatest reduction in "pressure" (e.g., memory usage, network congestion) is non-trivial. 

- **Conclusion**: This paradigm is better suited for highly dynamic runtime. For the static nature of our primitive-based solution, it is less appropriate than a global optimization approach.

### 2.2. Static Global Optimization

This approach treats the entire space-time trajectory of all CATs as a single point in a high-dimensional search space. The goal is to start with an initial, valid plan and iteratively modify it to find a better one within the strategy space defined by the primitives. This is the more promising direction for the CATFlow framework.

- **Concept**: Start with a simple, valid baseline strategy (e.g., pure data parallelism). Iteratively apply transformations to the plan to explore the neighborhood of the current solution, moving towards a solution with a lower estimated cost.

## 3. Challenges in Static Global Optimization

Despite being the more suitable approach, static optimization presents several significant challenges.

1.  **Vast and Complex Search Space**: The number of possible trajectories for each tensor is immense. The search space is not only large but also filled with constraints, making navigation difficult. High-quality solutions are likely sparse and hard to find.

2.  **Defining "Neighboring" Solutions**: A core part of iterative optimization is moving to a "nearby" solution. What constitutes a small change?
    - A single tensor's position at a single point in time?
    - Applying a different primitive (`split` instead of `copy`)?
    - Changing a parameter (e.g., the number of splits in a `split` operation)?
    The granularity of these "moves" is difficult to define, and each step presents a near-infinite number of choices.

3.  **Diverse Transformation Operations**: The "next step" in the search is not just a simple move. It can involve a wide variety of transformations, such as splitting a tensor, creating a replica, or adding/removing recomputation dependencies. This diversity complicates the design of the search algorithm.

4.  **Multi-Entity Coordination**: A change to one CAT's trajectory can have cascading effects on many others due to data dependencies. A greedy strategy that optimizes each CAT individually is unlikely to succeed. The search must coordinate changes across multiple CATs to ensure a global cost reduction.

## 4. Alternative Paradigm: Search by Progressive Space Reduction

An alternative to modifying a single, complete plan is to view the search problem as a process of **progressive space reduction**. This approach starts with the entire, vast strategy space defined by the initial primitives and systematically narrows it by adding new constraints until the space is reduced to a single, fully-defined parallelization plan.

- **Concept**: The search process is framed as a constraint satisfaction problem. The initial primitives define a solution space. The algorithm iteratively adds constraints, with each new constraint pruning large portions of the space that are deemed suboptimal or infeasible. The process concludes when enough constraints have been added to specify exactly one plan.

- **How It Works**:
    1.  **Initialization**: Start with the initial, lightly constrained space.
    2.  **Iterative Constraint Addition**: At each step, the algorithm chooses a new constraint to add. This choice is the core of the search. For example, it might decide:
        - The degree of pipeline parallelism is 4.
        - The tensor parallelism strategy for a specific stage is row-wise sharding.
        - A specific group of tensors should be offloaded to CPU memory between uses.
    3.  **Pruning**: After adding a constraint, the search space is re-evaluated, and all plans inconsistent with the new constraint are discarded.
    4.  **Termination**: The process repeats until the solution space is reduced to a single point (a concrete plan), or no more valid constraints can be added.

- **Potential Risks & Unconsidered Issues**:
    - **Ordering is Critical**: The sequence in which constraints are added heavily influences the outcome. An early, suboptimal decision (e.g., choosing the wrong number of pipeline stages) can permanently prune the globally optimal solution from the search space.
    - **Heuristic-Dependent**: The algorithm needs a powerful heuristic to decide which constraint to add next. This heuristic must estimate the "quality" of the remaining subspace after a potential constraint is added. Designing such a heuristic is extremely challenging, as it requires reasoning about a vast set of potential future solutions.
    - **Irreversibility and Backtracking**: Without a backtracking mechanism, the algorithm can easily get stuck in a poor region of the search space. However, implementing backtracking (e.g., removing a previously added constraint) adds significant complexity to the search algorithm.
    - **Cost of Evaluation**: Evaluating the impact of adding a constraint can be computationally expensive, as it may require a partial search or simulation of the remaining space.

- **Suggestions for Implementation**:
    - This approach could be implemented using search algorithms designed for constraint satisfaction, such as **Beam Search** or **A\* Search**.
    - In a Beam Search, the algorithm would keep track of the `k` most promising sets of constraints (partial solutions) at each step, providing a balance between greedy search and exhaustive exploration.
    - The cost function for the heuristic could be a "relaxed" performance model that provides a fast, optimistic lower bound on the best possible makespan within the remaining subspace.

This paradigm offers a structured way to decompose the complex search problem into a sequence of smaller decisions. However, its success is highly dependent on the quality of the decision-making heuristic at each step.

## 5. Proposed Solution: Hierarchical, Constraint-Guided Search

To tackle these challenges, we propose a search algorithm that is **hierarchical**, **constraint-guided**, and leverages the **unified nature of the CAT abstraction**. The core idea is to use a search method like simulated annealing or gradient-based optimization on a simplified, continuous representation of the strategy space.

### 5.1. Foundational Concepts

- **Symmetry Utilization**: Many parallelization strategies involve symmetry. For example, in data parallelism, the plan for each device is identical. The search algorithm should identify and exploit these symmetries. CATs with identical constraint sets can be grouped and transformed together, drastically reducing the effective size of the search space.

- **Continuity of the Strategy Space**: The data-monistic abstraction provides a more "continuous" or "smooth" strategy space compared to operator-centric views. For example, one can smoothly transition from data parallelism to tensor parallelism by gradually changing the `split` dimension and `assign` constraints on tensors. This smoothness makes iterative optimization methods more effective, as small changes to the plan are more likely to lead to small changes in performance.

下面大模型提出的方案：

### 5.2. A Two-Level Hierarchical Search Algorithm

We propose a two-level search process: a high-level search for the overall strategy and a low-level search for parameter tuning.

**Level 1: Strategy Search (Coarse-Grained)**

This level explores the "shape" of the parallelization strategy.

- **Representation**: We can represent a strategy by a set of active high-level constraints or by a configuration of the core primitives applied to major tensor groups (activations, weights, gradients).
- **Search Method**: A meta-heuristic algorithm like **Simulated Annealing** or **Genetic Algorithms** would be suitable here.
    - **State**: The current parallelization strategy (e.g., "3D Parallelism", "FSDP-style sharding").
    - **Move/Mutation**: A move consists of making a significant change to the strategy, such as:
        - Changing a `split` primitive on a weight tensor to a `copy` primitive.
        - Adding a `redirect` primitive to enable recomputation for a set of activations.
        - Changing the target of a `reduce` operation from all-to-all to a local reduction.
    - **Evaluation**: After each move, the low-level search (Level 2) is invoked to fine-tune the parameters of the new strategy, and the resulting cost is used to evaluate the move.

**Level 2: Parameter Search (Fine-Grained)**

Given a fixed strategy from Level 1, this level tunes its continuous and discrete parameters.

- **Representation**: For a given strategy, many parameters can be represented as a continuous vector. For example:
    - The degree of tensor parallelism can be a continuous variable that is discretized during cost evaluation.
    - The number of pipeline stages.
- **Search Method**: A **gradient-based method** can be used.
    - **Objective**: The cost function (estimated makespan).
    - **Challenge**: The cost function is not analytically differentiable with respect to the strategy parameters.
    - **Suggestion**: We can use techniques for optimizing non-differentiable functions:
        1.  **Finite Differences**: Approximate the gradient by slightly perturbing each parameter and observing the change in the cost function. This is simple but can be computationally expensive.
        2.  **Zeroth-Order Optimization / Gradient-Free Methods**: Use algorithms like Covariance Matrix Adaptation Evolution Strategy (CMA-ES) that do not require explicit gradient calculations and are effective for complex, non-convex landscapes.
        3.  **Surrogate Modeling**: Train a small neural network to approximate the cost function. This surrogate model can then be optimized using standard gradient descent, and its predictions can guide the search in the true cost landscape.

### 5.3. Algorithm Flow

1.  Start with a baseline strategy (e.g., data parallelism) and run the Level 2 search to optimize its parameters. This gives an initial cost.
2.  Enter the Level 1 loop (e.g., Simulated Annealing).
3.  Propose a new candidate strategy by applying a random transformation (mutation) to the current strategy.
4.  Run the Level 2 search on this new candidate strategy to find its optimal parameters and cost.
5.  Use the acceptance criteria of the meta-heuristic (e.g., the Metropolis-Hastings criterion in Simulated Annealing) to decide whether to accept the new strategy.
6.  Repeat until a termination condition is met (e.g., budget expires, no improvement is seen).

This hierarchical approach balances broad exploration of the strategy space with fine-grained optimization, providing a structured way to navigate the immense search space and discover novel, high-performance parallelization plans.

## 6. Future Direction: Optimization via Constraint Vector Modification

This chapter explores a more speculative but potentially powerful paradigm: representing the entire set of constraints on a CAT as a single **constraint vector** and using optimization techniques to modify this vector directly.

- **Concept**: The state of every CAT in the system is defined by its associated constraints. If we can encode these constraints into a numerical vector, the entire parallelization plan becomes a single, very large vector (the concatenation of all individual CAT vectors). The search problem is then transformed into finding the optimal point in this high-dimensional vector space.

- **Constraint Vector Representation**:
    The first challenge is to create a meaningful vector representation. A constraint vector for a single CAT would need to encode:
    - **Categorical Information**: The type of primitives applied (e.g., `split`, `copy`, `reduce`), the dimension of a split, the target of a reduction. This could be done with one-hot encoding.
    - **Numerical Information**: The number of ways a tensor is split, a specific time coordinate for an `order` constraint.
    - **Relational Information**: Dependencies on other CATs (e.g., for a `reduce` or `order` constraint). This is the most complex part, as it might involve pointers or IDs of other CATs.

    A simplified global vector could represent high-level choices for major tensor groups (weights, activations, gradients) rather than for individual CATs, making the vector smaller and more manageable.

- **Optimization as Vector Modification**:
    Once a vector representation exists, the goal is to find a modification `ΔV` to the current vector `V` such that `Cost(V + ΔV) < Cost(V)`. This framing naturally suggests gradient-based optimization. If we could compute the gradient of the cost function with respect to the constraint vector, `∇Cost(V)`, we could iteratively update the vector in the direction of steepest descent.

- **Potential Risks & Unconsidered Issues**:
    1.  **Non-Differentiable and Discontinuous Cost Function**: This is the most significant barrier. The relationship between the constraint vector and the final makespan is not a smooth, analytical function. A tiny change in the vector (e.g., flipping a single bit in a one-hot encoding to change a `split` to a `copy`) can result in a completely different strategy and a large, discontinuous jump in the cost. Standard gradient descent is not applicable on such a landscape.
    2.  **Validity of Modified Vectors**: The search space is not a simple Euclidean space. Most points in the vector space do not correspond to valid, physically possible parallelization plans. A random modification to a valid vector will almost certainly produce an invalid one (e.g., a vector that implies a tensor is in two places at once). The optimization must be heavily constrained to only explore the manifold of valid plans.
    3.  **High Dimensionality**: The concatenated vector for a real-world model would be extremely high-dimensional, making any search difficult and computationally expensive (the "curse of dimensionality").
    4.  **Lack of Interpretability**: The gradient, even if it could be approximated, might not provide an interpretable direction for improvement. What does it mean to apply a small update to a one-hot encoded vector? This requires a method to project the updated vector back onto the nearest valid, discrete choice.

- **Suggestions for Application**:
    - **Gradient-Free Optimization**: Instead of gradient descent, this vector representation is well-suited for gradient-free methods like those mentioned in Chapter 5 (e.g., CMA-ES, genetic algorithms). These methods do not assume differentiability and are designed to work on black-box objective functions. A "mutation" in a genetic algorithm could be a carefully designed perturbation of the constraint vector.
    - **Surrogate Modeling**: This is a very strong candidate for this paradigm. A neural network could be trained as a surrogate model to map a (potentially simplified) constraint vector `V` to a predicted cost. This surrogate *is* differentiable, so you could compute gradients with respect to its input vector. These gradients would then guide the search for promising new vectors to evaluate on the true cost model.
    - **Local Search Refinement**: The constraint vector idea could be used for fine-tuning an existing, high-quality plan. Starting from a good solution, the algorithm could explore small, valid perturbations of its vector representation to see if minor local changes (e.g., slightly shifting a communication event time) can yield further improvements.

In summary, while direct gradient descent on a constraint vector is likely infeasible, the concept of vectorizing the strategy space is a powerful abstraction that opens the door to applying sophisticated black-box optimization and surrogate modeling techniques.

## Integrating Refinement Engine and Perturbation Engine

In CATFlow's automated parallelization search algorithm, the Refinement Engine and Perturbation Engine can complement each other, corresponding to two classic reasoning and optimization paradigms.

### 1. Refinement Engine: Systematic Constraint Propagation

The Refinement Engine adopts a "broad-to-narrow" strategy, suitable for global, systematic exploration of the solution space:

- **Initialization**: Each variable (e.g., tensor space-time trajectory, distribution strategy) has a full domain representing all possible values.
- **Constraint Propagation**: Based on model structure, hardware resources, and known constraints, variable domains are pruned. For example, a tensor may be excluded from a device due to memory limits, or certain parallel strategies may be mutually exclusive.
- **Convergence Check**: Propagation and pruning continue until all variables are uniquely determined or domains cannot be further reduced.
- **Advantages**: Systematically eliminates illegal or inefficient plans, ideal for the initial stage to shrink the search space and ensure feasibility and global consistency.

### 2. Perturbation Engine: Efficient Response and Local Optimization

The Perturbation Engine uses a "start from a complete state and respond to changes" strategy, ideal for efficient local optimization based on an existing feasible solution:

- **Initialization**: Start from a complete parallelization plan output by the Refinement Engine.
- **Perturbation and Propagation**: When a variable (e.g., a layer's parallel strategy or tensor placement) is modified, the Perturbation Engine quickly adjusts affected variables to restore consistency.
- **Use Cases**: Suitable for neighborhood perturbations in heuristic searches (e.g., simulated annealing, genetic algorithms) or instant feedback during interactive parameter tuning.
- **Advantages**: Efficient and responsive, avoids global recomputation, suitable for large models and complex constraints.

### 3. Collaboration and Integration

- **Phased Combination**: Use the Refinement Engine for global constraint propagation and space pruning to obtain high-quality initial solutions, then apply the Perturbation Engine for fine-grained optimization in their neighborhoods.
- **Dynamic Switching**: Switch to the Refinement Engine when encountering global constraint conflicts or needing large-scale adjustments; use the Perturbation Engine for local tuning and responding to external changes.
- **Unified Abstraction**: Both engines can be implemented based on CATFlow's constraint-variable model, differing in operation granularity and propagation scope.

### 4. Application Examples

- **Automated Parallel Search**: Use the Refinement Engine to prune illegal parallel strategy combinations, then apply the Perturbation Engine for parameter perturbation and local adjustment to improve performance.
- **Interactive Tuning**: When a user modifies a parallel parameter, the Perturbation Engine provides instant feedback without global recomputation.

### 5. Summary

The combination of Refinement Engine and Perturbation Engine provides CATFlow's parallelization search with a framework that is both systematic and highly responsive. The former ensures global consistency and feasibility, while the latter boosts search efficiency and interactive experience. Their synergy can significantly enhance the intelligence and practical usability of automated parallelization.

## 6. An Idealized Experimental Model

This experimental model aims to distill the complex optimization problem above into a simplified mathematical model that highlights core issues such as smoothness and the feasibility of greedy strategies.

In this model, consider a system with a single constraint: initially, there are n containers in a row, each containing a different number of balls. At each step, every ball moves to the adjacent container with fewer balls; if the current container already has the fewest, the ball stays. After several steps, the goal is for all containers to have roughly equal numbers of balls.
