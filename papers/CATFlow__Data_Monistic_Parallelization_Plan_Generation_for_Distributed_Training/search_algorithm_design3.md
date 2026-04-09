文章结构建议：

**基本结构**：
1. 明确阐述该搜索算法在设计时需要关注的核心问题，以及实际应用中可能遇到的主要难点。
2. 针对上述问题和难点，提出基本的解决思路或应对策略。
3. 给出一个完整的搜索算法示例，并详细解释其关键步骤和设计要点。

**优化方向**：
- 说明将数据单一性（data-monistic）的抽象引入到constraint-aware搜索中的优势，突出其对简化约束处理和提升算法效率的作用。
- 进一步解释“可搜索性”（searchability）的动机和意义，尤其是和operator-centric方法的对比。
- 尝试提出一种尽量统一、简洁的机制，以巧妙地应对搜索中的复杂难题，避免针对不同情况采用繁琐的特殊处理。

由于搜索部分是本文的三大创新点之一，因此建议在本节中突出展示独特的思路或方法论。即使读者尚未完全理解具体实现，也能从整体设计理念中感受到其新颖性和价值。通过强调方法背后的核心思想和创新视角，使本节内容具有启发性和吸引力。（需要注意的是，提出独特视角本身并不难，真正的挑战在于把握好创新的尺度：在一个已有大量深入研究和广泛共识的优化领域，越是强调方法的新颖性和颠覆性，就越需要充分论证其合理性。这往往要求对现有认知进行深入分析和解释，而这与篇幅有限的实际情况形成矛盾。因此，在展示创新时，应在突出核心思想的同时，兼顾论证的深度与表达的简洁。）

也就是说，还是要尽量说明我们的设计为什么好。


# 一些碎片

From a search perspective, this constrained space can be explored along several axes. The application of primitives defines the fundamental structure and boundaries of the search. For example, primitives like \texttt{split}, \texttt{copy}, or \texttt{reduce} govern the creation and consolidation of CATs, fundamentally altering the number and nature of the entities being scheduled. Within this structure, the search algorithm can further explore variations by: (1) adjusting the space-time domain for each event associated with a CAT, effectively tuning its placement and timing; (2) modifying the set of events for a given CAT, such as by introducing a prefetch event; and (3) altering the constraints between CATs, for instance by adding or removing an \texttt{order} or \texttt{redirect} dependency to explore different execution schedules or recomputation strategies. By systematically defining and constraining this multi-faceted space, \name{} transforms the intractable problem of finding an optimal plan into a structured and solvable search problem.


While the data-monistic abstraction offers exceptional flexibility and completeness, these very characteristics introduce formidable challenges for the search process. The core difficulty stems from the search space being a dynamic and structurally complex landscape rather than a static set of tunable parameters. 

# 背景铺垫部分

先介绍策略搜索这个问题（在Strategy space中找到一个尽量improved的策略），并指出他是很难的，具体说出难在哪里（现有的方法大多是在给出一个strategy template的情况下，搜索template给出的参数。而我们要做的，是在primitive定义了一个策略空间后，多样性的（甚至是人设计之外的）策略，只要是满足策略空间的约束，都会存在于这个空间中，都可以被搜索到）。然后再单独开始一部分，指出data-monistic的抽象因为 **拓扑同质性（Topological Homogeneity）**、**优化算子封闭性（Operator Closure）**、**启发式信息一致性（Heuristic Consistency）** 能更好的解决strategy-level的搜索。

The primary challenge in distributed training is not merely tuning parameters but performing true strategy search—discovering a high-performance parallelization plan from a vast, complex strategy space. Most existing frameworks operate by first defining a fixed strategy template, such as 3D parallelism, and then conducting a parameter search within that template to determine variables like partition sizes or device mappings. This fundamentally limits the search to human-predefined designs. In contrast, our goal is to enable the search for diverse and even novel strategies that emerge from a space defined by a set of primitives. Any strategy satisfying the space's constraints can be discovered, moving beyond parameter tuning to genuine strategy-level exploration.

However, performing genuine strategy-level exploration is exceptionally challenging due to the heterogeneous and combinatorial nature of the operator-based search space. A parallelization strategy is defined by a complex combination of fundamentally different types of decisions: how to partition tensors (e.g., data or tensor parallelism), where to place computations, when to schedule operations, and whether to recompute data or fetch it from remote memory. This creates a disjointed and rugged search landscape where a minor change in one decision type, such as switching from recomputation to data fetching, can lead to a drastically different and incomparable solution structure. Consequently, traditional search algorithms cannot rely on a unified optimization approach (like gradient descent) and must instead use complex, hand-crafted heuristics to navigate a space composed of discrete, non-interchangeable choices, making the systematic discovery of novel strategies intractable.

Our data-monistic abstraction is uniquely suited to address this complex strategy search problem by enhancing the "searchability" of the solution space. This improvement stems from two key properties. First, it establishes **Topological Homogeneity**. Operator-based abstractions are typically composed of discrete and qualitatively different operation types. By representing any strategy as a unification of CATs, transitions between vastly different high-level strategies can be decomposed into a sequence of small, incremental modifications to the underlying CAT abstraction. This creates a connected landscape where strategies are not isolated points but interconnected neighbors, making the space far more amenable to systematic exploration by local search algorithms. Second, it provides **Heuristic Consistency**. By defining scheduling constraints on fine-grained CATs rather than coarse-grained operators, the abstraction increases the number of decision variables. This higher-dimensional space creates more "escape routes" from local optima, enabling more consistent and reliable greedy search paths. Together, these properties transform the intractable problem of strategy discovery into a structured and tractable optimization task.

# 提出基本的算法思路

Building upon these properties, we propose a scheduling model inspired by potential field methods, which transforms the scheduling problem into a search for the minimum energy state within a multi-dimensional potential field. First, we represent the constraints on all event points of a CAT using a *constraint vector*, where each element represents a space-time domain of a event point. The search process manifests as modifications to this constraint vector, which include: (1) specifying concrete space-time intervals, (2) altering the vector's dimensionality by adding or removing event points, and (3) generating new constraint vectors for newly created CATs.

For any given parallel plan, we define a system-wide "potential energy." This energy is evaluated based on the "pressure" on each device, where high pressure corresponds to resource oversubscription (e.g., exceeding memory capacity) or poor performance. Our heuristic search algorithm then guides each modification to a CAT's constraint vector in the direction that yields the steepest descent in the overall potential energy. To offer an intuitive analogy, one can imagine the "pressure" on each device as the "water level" at different locations in a pond. Each CAT acts as an independent stream of water, naturally flowing from high-pressure regions to low-pressure ones, thereby lowering the system's overall potential energy.


# 给出并解释具体的算法

To formalize this search process, we employ a structured local search algorithm guided by the principle of potential energy minimization. The algorithm iteratively refines a parallelization plan by exploring neighboring solutions and accepting modifications that reduce the system's overall potential. The pseudocode below outlines this framework.

```latex
\begin{algorithm}[H]
\caption{Structured Local Search for Strategy Discovery}
\label{alg:structured_search}
\begin{algorithmic}[1]
\State \textbf{Input:} Initial parallelization plan $S_{initial}$
\State \textbf{Output:} Optimized parallelization plan $S_{final}$
\State
\State $S_{current} \gets S_{initial}$
\State $P_{current} \gets \text{CalculatePotential}(S_{current})$
\State
\For{$i = 1$ to $max\_iterations$}
    \State $improved \gets \text{false}$
    \State $M_{candidates} \gets \text{GetCandidateMoves}(S_{current})$
    \State
    \For{each move $m$ in $M_{candidates}$}
        \State $S_{neighbor} \gets \text{ApplyMove}(S_{current}, m)$
        \State $P_{neighbor} \gets \text{CalculatePotential}(S_{neighbor})$
        \State
        \If{$P_{neighbor} < P_{current}$}
            \State $S_{current} \gets S_{neighbor}$
            \State $P_{current} \gets P_{neighbor}$
            \State $improved \gets \text{true}$
            \State \textbf{break} \Comment{First-improvement strategy}
        \EndIf
    \EndFor
    \State
    \If{not $improved$}
        \State \textbf{break} \Comment{Local minimum reached}
    \EndIf
\EndFor
\State
\State \textbf{return} $S_{current}$
\end{algorithmic}
\end{algorithm}
```

The algorithm operates as follows. It begins with an initial, potentially suboptimal parallelization plan, $S_{initial}$. The core of the algorithm is an iterative loop that seeks to find a better plan in each step.

The `CalculatePotential` function (line 5, 13) is central to this process, acting as the objective function. It evaluates a given plan $S$ and computes a scalar "potential energy" value. This value quantifies the plan's inefficiency, primarily by measuring the extent of resource oversubscription. For instance, it calculates the total memory overflow across all devices by summing up the difference between the memory demand of CATs assigned to a device and the device's capacity. A lower potential signifies a more balanced and efficient plan.

In each iteration, the `GetCandidateMoves` function (line 9) heuristically generates a list of promising modifications. Instead of exploring all possible moves, it focuses on "problem areas." For example, it identifies devices with high potential (e.g., memory overflow) and proposes moves for the CATs on those devices, such as migrating a CAT to a less loaded device (`move` operator) or breaking a large CAT into smaller ones (`split` operator). This targeted approach makes the search more efficient.

The algorithm then iterates through these candidate moves, applies each one to generate a `neighbor` plan using `ApplyMove` (line 12), and evaluates its potential. Following a first-improvement strategy, it immediately accepts the first move that results in a lower potential, updates the current plan, and restarts the iteration. If no candidate move yields an improvement, the algorithm concludes that it has reached a local minimum and terminates.
