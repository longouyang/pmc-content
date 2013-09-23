% Patterns of Inference

# Causal Dependence
<!--A probabilistic program describes a stochastic procedure for generating possible worlds, and each world can be characterized by the sequence of individual stochastic choices, or random function calls, made in deriving it.  The individual functions comprising a program (or more precisely, the individual function calls comprising a specific
run of the program) can be given a natural partial order based on their dependency structure.  In running the program, some functions can be evaluated right at the start; their output does not depend on the behavior of any other functions.  Other functions depend on the outputs of earlier function evaluations; the random choices made in the later functions will tend to come out differently depending on the random outputs of earlier functions.  Still other functions do not even properly exist when the program begins running; they will be created on the fly by other higher-order functions in the course of evaluation.  We can often gain insight into the knowledge a program encodes, and the patterns of conditional reasoning it supports, by analyzing this dependency structure. -->

Probabilistic programs encode knowledge about the world in the form of causal models, and it is useful to understand how their function relates to their structure by thinking about some of the intuitive properties of causal relations.  Causal relations are local, modular, and directed.  They are *modular* in the sense that any two arbitrary events in the world are most likely causally unrelated, or independent.  If they are related, or dependent, the relation is only very weak and liable to be ignored in our mental models.  Causal structure is *local* in the sense that many events that are related are not related directly: They are connected only through causal chains of several steps, a series of intermediate and more local dependencies.  And the basic dependencies are *directed*: when we say that A causes B, it means something different than saying that B causes A.The *causal influence* flows only one way along a causal relation&mdash;we expect that manipulating the cause will change the effect, but not vice versa&mdash;but *information* can flow both ways&mdash;learning about either event will give us information about the other.

Let's examine this notion of "causal dependence" a little more carefully.  What does it mean to believe that A depends causally on B?  Viewing cognition through the lens of probabilistic programs, the most basic notions of causal dependence are in terms of the structure of the program and the flow of evaluation (or "control") in its execution.
We say that expression A causally depends on expression B if it is necessary to evaluate B in order to evaluate A. (More precisely, expression A depends on expression B if it is ever necessary to evaluate B in order to evaluate A.) Note that this causal dependence order is weaker than a notion of ordering in time&mdash;one expression might happen to be evaluated before another in time (for instance the first of two arguments to a function before the second), but without the second expression requiring the first. (This notion of causal dependence is roughly the same as the notion of [control flow dependence](http://en.wikipedia.org/wiki/Dependence_analysis) in the programming language literature.)

For example, consider a simpler variant of our medical diagnosis scenario:

~~~~ {.mit-church}
(define samples
  (mh-query 200 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.01)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

   (list cold lung-disease)

   cough
  )
)
(hist (map first samples) "cold")
(hist (map second samples) "lung-disease")
(hist samples "cold, lung-disease")
~~~~

Here, `cough` depends causally on both `lung-disease` and `cold`, while `fever` depends causally on `cold` but not `lung-disease`.  We can see that `cough` depends causally on `smokes` but only indirectly: although `cough` does not call `smokes` directly, in order to evaluate whether a patient coughs, we first have to evaluate the expression `lung-disease` that must itself evaluate `smokes`. <ref>We haven't made the notion of "direct" causal dependence precise: do we want to say that `cough` depends directly on `cold`, or only directly on the expression `(or (and cold (flip 0.5)) ...`? This can be resolved in several ways that all result in similar intuitions. For instance, we could first re-write the program into a form where each intermediate expression is named, so called A-normal form, then say direct dependence is when one expression immediately includes the name of another.</ref> The causal dependence structure is not always immediately clear from examining a program, particularly where there are complex functions calls; a useful way of diagnosing causal dependence, which is even sometimes taken as the definition, is by *intervention* on the program&mdash;see the sidebar.

<style classes="bg-gray">
<collapse name="Detecting Dependence Through Intervention">
Another way to detect (or according to some philosophers, such as Jim Woodward, to *define*) causal dependence is more functional, in terms of "difference making": If we manipulate A, does B tend to change? By *manipulate* here we don't mean assume in the sense of `query`. Instead we mean actually edit, or *intervene on*, the program in order to make an expression have a particular value independent of its (former) causes. If setting A to different values in this way changes the distribution of values of B, then B causally depends on A. This method is known in the causal Bayesian network literature as the "do operator" or graph surgery (Pearl, 1988).  It is also the basis for interesting theories of counterfactual reasoning by Pearl and colleagues (Halpern, Hitchcock and others).

For example, this code represents whether a patient is likely to have a cold or a cough *a priori*, conditioned on no observations:

~~~~ {.mit-church}
(define samples
  (mh-query 200 1
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.01)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

   (list cough cold)

   true
  )
)
(hist (map first samples) "cough")
(hist (map second samples) "cold")
~~~~
See what happens if we give our hypothetical patient a  cold, e.g., by exposing him to a strong cocktail of cold viruses.  Formally we implement this not  by observing `cold` (conditioning on having a cold in the Church query) but by forcing it in the definition: first do `(define cold true)`, then do `(define cold false)`.

~~~~ {.mit-church}
(define samples
  (mh-query 200 1
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold true) ;we intervene to make cold true.

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.01)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

   (list cough cold)

   true
  )
)
(hist (map first samples) "cough")
(hist (map second samples) "cold")
~~~~

You should see that the distribution on `cough` changes: coughing becomes more likely if we know that a patient has been given a cold by external intervention.  But the reverse is not true.  Try forcing the patient to have a  cough (e.g., with some unusual drug or by exposure to some cough-inducing dust) by writing `(define cough true)`: the distribution on `cold` is unaffected.
We have captured a familiar fact: treating the symptoms of a disease directly doesn't cure the disease (taking cough medicine doesn't make your cold go away), but treating the disease *does* relieve the symptoms.

Verify in the Church query above that the method of manipulation works also to identify causal relations that are only indirect: e.g., force a patient to smoke and show that it increases their probability of coughing, but not vice versa.  If we are given a program representing a causal model, and the model is simple enough, it is straightforward to read off causal dependencies from the structure of the program. However, the notion of causation as difference-making may be easier to compute in much larger, more complex models -- and it does not require an analysis of the program code.  As long as we can modify (or imagine modifying) the definitions in the program and can `query` the resulting model, we can compute whether two events or functions are causally related by the difference-making criterion.
</collapse>
</style>

There are several special situations that are worth mentioning. In some cases, whether expression A requires expression B will depend on the value of some third expression C.  For example, here is a particular way of writing a noisy-logical AND gate:

~~~~ {.norun}
(define C (flip))
(define B (flip))
(define A (if C (if B (flip 0.85) false) false))
~~~~

A always requires C, but only evaluates B if C returns true.  Under the above definition of causal dependence A depends on B (as well as C). However, one could imagine a more fine-grained notion of causal dependence that would be useful here: we could say that A depends causally on B only in certain *contexts* (just those where C happens to return true and thus A calls B).

Another nuance is that an expression that occurs inside a function body may get evaluated several times in a program execution. In such cases it is useful to speak of causal dependence between specific evaluations of two expressions. (However, note that if a specific evaluation of A depends on a specific evaluation of B, then any other specific evaluation of A will depend on *some* specific evaluation of B. Why?)
<!-- Research Note: This should allow us to define the difference between causes and enabling conditions, in terms of computational efficiency.  It should also allow us to define "actual cause" in terms of the dependence of specific function calls on other specific function calls in a particular evaluation of the program. -->

# Statistical Dependence

One often hears the warning, "correlation does not imply causation".  By "correlation" we mean a different kind of dependence between events or functions, sometimes called "statistical dependence" to distinguish it from "causal dependence". We say that A and B are statistically dependent, if learning about A tells us something about B, and vice versa.  In the language of Church: using `query` to assert something about A changes the value expected for B. (If using query to condition on A changes B, then conditioning on B also changes A. Why?) Statistical dependence is a *symmetric* relation between events referring to how information flows between them when we observe or reason about them.
The fact that we need to be warned against confusing these notions suggests they are related, and indeed, they are.  (One might even say they are "causally related", in the sense that causal dependencies cause, or give rise to, statistical dependencies.)  In general, if A causes B, then A and B will be statistically dependent.

Using the above example (press Reset if necessary to undue your last changes), verify that this sort of statistical dependence holds between `cough` and `cold`, and is symmetric unlike causal dependence.  Verify also that statistical dependence holds symmetrically for events that are connected by an indirect causal chain, such as `smokes` and `coughs`.

Correlation is not just a symmetrized version of causality.  Two events may be statistically dependent even if there is no causal chain running between them, as long as they have a common cause (direct or indirect).  Two expressions in a Church program can be statistically dependent if one calls the other, directly or indirectly, *or* if they both at some point in their evaluation histories refer to some other expression (a "common cause").  In the example above, `cough` and `fever` are not causally dependent but they are statistically dependent, because they both depend on `cold`; likewise for `chest-pain` and `shortness-of-breath` which both call `lung-disease`.  Here we can read off these facts from the program definitions, but more generally all of these relations can be diagnosed by reasoning using `query`.  (Show this in the above example.)

Successful learning and reasoning with causal models typically depends on exploiting the close coupling between causation and correlation.  This is because causal relations are typically unobservable, while correlations are observable from data.  Noticing patterns of correlation is thus often the beginning of causal learning, or discovering what causes what.  With a causal model already in place, reasoning about the statistical dependencies implied by the model allows us to predict many aspects of the world not directly observed from those aspects we do observe.
<!-- When the time comes to act, though -- to manipulate the objects in our world -- then we must be careful not to confuse statistical and causal dependence.-->

# Graphical Notations for Dependence

*Graphical models* are an extremely important idea in modern machine learning: a graphical diagram is used to represent the direct dependence structure between random choices in a probabilistic model. A special case are *Bayesian networks*, in which there is a node for each random variable (and expression in our terms) and a link between two nodes if there is a direct conditional dependence between them (a direct causal dependence in our terms). The sets of nodes and links define a *directed acyclic graph* (hence the term graphical model), a data structure over which many efficient algorithms can be defined. Each node has a *conditional probability table* (CPT), which represents the conditional probability distribution of that node, given its parents. The joint probability distribution over random variables is given by the product of the conditional distributions for each variable in the graph.

The figure below defines a Bayesian network for the medical diagnosis example. The graph contains a node for each random variable assignment (`define` statement) in our church program, with links to that node from each variable that appears in the assignment expression. There is a CPT for each node, with a column for each value of the random variable, and a row for each combination of values for its parents.

<img src='images/Med-diag-bnet1.jpg' width=600 />

Simple generative models will have a corresponding graphical model that captures all of the dependencies (and *in*dependencies) of the model, without capturing the precise *form* of these functions. For example, while the graphical model shown above faithfully represents the probability distribution encoded by the church program, it captures the *noisy-OR* form of the causal dependencies only implicitly. As a result, the CPTs provide a less compact representation of the conditional probabilities than the church model. For instance, the CPT for `cough` specifies 4 parameters -- one for each pair of values of `lung-disease` and `cold` (the second entry in each row is determined by the constraint that the conditional distribution of `cough` must sum to 1). In contrast, the `define` statement for `cold` in church specifies only 3 parameters: the base rate of `cough`, and the strength with which `lung-disease` and `cold` cause `cough`. This difference becomes more pronounced for noisy-OR relations with many causes -- the size of the CPT for a node will be exponential in the number of parents, while the number of terms in the noisy-OR expression in church for that node will be linear in the number of causal dependencies (why?). As we will see, this has important implications for the ability to learn the values of the parameters from data.

More complicated generative models, which can be expressed as probabilistic programs, often don't have such a graphical model (or rather they have many approximations, none of which captures all independencies). We will see examples when we study recursive models later on.

# From *A Priori* Dependence to Conditional Dependence

The relationships between causal structure and correlational structure, or statistical dependence, become particularly interesting and subtle when we look at the effects of further observations or assumptions.  Events that are statistically dependent *a priori* (sometimes called *marginally* dependent) may become independent when we condition on some other observation; this is often called *screening off*, or sometimes *context-specific independence*.  Also, events that are statistically independent *a priori* (marginally independent) may become dependent when we condition on other observations; this is known as *explaining away*.  The dynamics of screening off and explaining away are extremely important for understanding patterns of inference -- reasoning and learning -- in probabilistic models.

<style classes="bg-gray">
<collapse name="Basic patterns of statistical dependence">

![](images/Marg-dep1.jpg)

![](images/Cond-dep1.jpg)

</collapse>
</style>



## Screening off

"Screening off" refers to a pattern of statistical inference that is quite common in both scientific and intuitive reasoning.  If the statistical dependence between two events A and B is only indirect, mediated strictly by one or more other events C, then conditioning on (observing) C should render A and B statistically independent.  This can occur if A and B are connected by one or more causal chains, and all such chains run through the set of events C, or if C comprises one or more common causes of A and B.  As an example in our simple medical scenario, `smokes` is correlated with (statistically dependent on) several symptoms, `cough`, `chest-pain`, and `shortness-of-breath`, due to the causal chain between them mediated by `lung-disease`.  We can see this easily by conditioning on these symptoms and querying `smokes`:

~~~~ {.mit-church}
(define samples
  (mh-query 200 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.001)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

     smokes

     (and cough chest-pain shortness-of-breath)
  )
)
(hist samples "smokes")
~~~~

The conditional probability of `smokes` is much higher than the base rate, 0.2, because observing all these symptoms gives strong evidence for smoking.  See how much evidence the different symptoms contribute by dropping them out of the conditioning set.  For instance, condition the above query on `(and cough chest-pain)`, or just `cough`; you should observe the probability of `smokes` decrease as fewer symptoms are observed.

Now, suppose we condition also on knowledge about the function that mediates these causal links: `lung-disease`.  Is there still an informational dependence between these various symptoms and `smokes`, conditioned on knowing that `lung-disease` is `true` or `false`?  In the query below, try adding and removing various symptoms (`cough`, `chest-pain`, `shortness-of-breath`) but maintaining the observation `lung-disease` (note: it can be tricky to diagnose statistical *in*dependence using mh-query, since natural variation due to random sampling can look like differences between conditions; using exact-query can help here, though it is slower):

~~~~ {.mit-church}
(define samples
  (mh-query 500 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.001)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

     smokes

     (and lung-disease
          (and cough chest-pain shortness-of-breath)
      )
  )
)
(hist samples "smokes")
~~~~

Try the same experiment of varying the symptom patterns for `cough`, `chest-pain`, `shortness-of-breath` while maintaining the observation `(not lung-disease)`:

~~~~ {.mit-church}
(define samples
  (mh-query 500 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.001)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

     smokes

     (and (not lung-disease)
          (and cough chest-pain shortness-of-breath)
      )
  )
)
(hist samples "smokes")
~~~~
You should see an effect of whether or not the patient has lung disease on conditional inferences about smoking -- a person is judged to be substantially more likely to be a smoker if they have lung disease than otherwise -- but there is no separate effects of chest pain, shortness of breath or cough, over and above the evidence provided by knowing whether the patient has lung-disease.  We say that the intermediate variable lung disease *screens off* the root cause (smoking) from the more distant effects (coughing, chest pain and shortness of breath).

Screening off as defined here is a purely statistical phenomenon.  When we observe C, the event(s) that mediate an indirect causal relation between A and B, A and B are still causally dependent in our model of the world: it is just our beliefs about the states of A and B that become uncorrelated.  There is also an analogous causal phenomenon.  If we can actually manipulate or *intervene* on the causal system, and set the value of C to some known value, then A and B become both statistically and causally independent.  Try this out by editing the above Church program: `(define lung-cancer true)`.

## Explaining away

"Explaining away" <ref>Pearl, J. Probabilistic Reasoning in Intelligent Systems, San Mateo: Morgan Kaufmann, 1988.</ref> refers to a complementary pattern of statistical inference which is somewhat more subtle than screening off. If two events A and B are statistically (and hence causally) independent, but they are both causes of one or more other events C, then conditioning on (observing) C can render A and B statistically dependent.  As with screening off, we are only talking about inducing statistical dependence here, not causal dependence: when we observe C, A and B remain causally independent in our model of the world; it is just our states of knowledge about A and B that become correlated.
<!-- (Unlike with screening off, however, there is no direct causal analog of explaining away.  If we actually intervene on the causal system to set C to some known value, we do *not* induce a causal dependence between A and B.)-->

Here is a concrete example in our medical scenario.  Having a cold and having lung disease are *a priori* independent both causally and statistically.  But because they are both causes of coughing if we observe `cough` then `cold` and `lung-disease` become statistically dependent.  That is, learning something about whether a patient has `cold` or `lung-disease` will, in the presence of their common effect `cough`, convey information about the other condition.  We say that `cold` and `lung-cancer` are marginally (or *a priori*) independent, but *conditionally dependent* given `cough`.

To illustrate, observe how the probabilities of `cold` and `lung-disease` change when we observe `cough` is true:

~~~~ {.mit-church}
(define samples
  (mh-query 500 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.001)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

   (list cold lung-disease)

   cough
  )
)
(hist (map first samples) "cold")
(hist (map second samples) "lung-disease")
(hist samples "cold, lung-disease")
~~~~
Both cold and lung disease are now far more likely that their baseline probability: the probability of having a cold increases from 2% to around 40%; the probability of having lung disease increases from 1 in a 1000 to a few percent.  Given a cough, `cold` is the much more likely explanation simply because it was much more probable a priori.

Now suppose we learn that the patient does *not* have a cold.

~~~~
(define samples
  (mh-query 500 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.001)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

   (list cold lung-disease)

   (and cough (not cold))
  )
)
(hist (map first samples) "cold")
(hist (map second samples) "lung-disease")
(hist samples "cold, lung-disease")
~~~~
The probability of having lung disease increases dramatically.  If instead we had observed that the patient does have a cold, the probability of lung cancer returns to its very low base rate of 1 in a 1000.

~~~~ {.mit-church}
(define samples
  (mh-query 500 100
    (define smokes (flip 0.2))

    (define lung-disease (or (flip 0.001) (and smokes (flip 0.1))))
    (define cold (flip 0.02))

    (define cough (or (and cold (flip 0.5)) (and lung-disease (flip 0.5)) (flip 0.001)))
    (define fever (or (and cold (flip 0.3)) (flip 0.01)))
    (define chest-pain (or (and lung-disease (flip 0.2)) (flip 0.01)))
    (define shortness-of-breath (or (and lung-disease (flip 0.2)) (flip 0.01)))

   (list cold lung-disease)

   (and cough cold)
  )
)
(hist (map first samples) "cold")
(hist (map second samples) "lung-disease")
(hist samples "cold, lung-disease")
~~~~

This is the conditional statistical dependence between lung disease and cold, given cough: Learning that the patient does in fact have a cold "explains away" the observed cough, so the alternative of lung disease decreases to a much lower value - roughly back to its 1 in a 1000 in the general population.
If on the other hand, we had learned that the patient does not have a cold, so the most likely alternative to lung disease is *not* in fact available to "explain away" the observed cough, that raises the conditional probability of lung disease dramatically.  As an exercise, check that if we remove the observation of coughing, the observation of having a cold or not having a hold has no influence on our belief about lung disease; this effect is purely conditional on the observation of a common effect of these two causes.

Consider (in the above example) a patient with a cough who also smokes, you will find that cold and lung disease are roughly equally likely explanations.
Or add the observation that the patient has chest pain, so lung disease becomes an even more probable condition than having a cold.  These are the settings where explaining away effects will be strongest.  Modify the above program to observe that the patient either has a cold or does not have a cold, in addition to having a cough, smoking, and perhaps having chest-pain.
<!--
E.g., compare these conditioners:

  (and smokes cough) with (and smokes cough cold) or
                          (and smokes cough (not cold))

  (and smokes chest-pain cough) with (and smokes chest-pain cough cold) or
                                     (and smokes chest-pain cough (not cold))
-->
Notice how far up or down knowledge about whether the patient has a cold can push the conditional belief in having lung disease.

Explaining away effects can be more indirect.  Instead of observing the truth value of `cold`, a direct alternative cause of `cough`, we might simply observe another symptom that provides evidence for `cold`, such as `fever`.  Compare these conditioners with the above church program to see an "explaining away" conditional dependence in belief between `fever` and `lung-disease`.

~~~~ {.norun}
(and smokes chest-pain cough) with (and smokes chest-pain cough fever) or
                                   (and smokes chest-pain cough (not fever))
~~~~

In this case, finding out that the patient either does or does not have a fever makes a crucial difference in whether we think that the patient has lung disease... even though fever itself is not at all diagnostic of lung disease, and there is no causal connection between them.
<!--
This is an important point, but I'm not sure how to work it in.  It should also be made more general, beyond linguistics:

In models in cognitive science in general and linguistics in particular this kind of independence is often reflected in a certain *modularity* in the system. For instance, in generative models of language it is often assumed that phonology and semantics don't interact directly, but only through the syntax.

-->

We can express the general phenomenon of explaining away with the following schematic Church query:

~~~~ {.norun}
(query
  (define a ...)
  (define b ...)
  ...
  (define data (... a... b...))

  b

  (and (equal? data some-value) (equal? a some-other-value)))
~~~~

We have defined two independent variables `a` and `b` both of which are used to define the value of our data. If we condition on the data and `a` the posterior distribution on `b` will now be dependent on `a`: observing additional information about `a` changes our conclusions about `b`.

The most typical pattern of explaining away we see in causal reasoning is a kind of *anti-correlation*: the probabilities of two possible causes for the same effect increase when the effect is observed, but they are conditionally anti-correlated, so that observing additional evidence in favor of one cause should lower our degree of belief in the other cause.  However, the coupling in belief
states induced by conditioning on common effects is not always an anti-correlation.  That depends on the nature of the interaction between the causes.

Explaining away takes the form of an anti-correlation when the causes interact in a roughly disjunctive or additive form: e.g., the effect tends to happen if cause A or cause B produce it; or the effect happens if the sum of cause A's and cause B's continuous influences exceeds some threshold.  The following simple mathematical examples show some of the other possibilities.  Suppose we condition on observing the sum of two integers drawn uniformly from 0 to 9.

~~~~
(define take-sample (lambda ()
  (rejection-query
    (define A (random-integer 10))
    (define B (random-integer 10))

    (pair A B)

    (equal? (+ A B) 9)
  )))
(define samples (repeat 500 take-sample))
(hist samples "A, B")

(scatter samples "A and B, conditioned on A + B = 9")
~~~~

This gives perfect anti-correlation in conditional inferences for A and B.  But suppose we instead condition on observing that A and B are equal.

~~~~
(define take-sample (lambda ()
  (rejection-query
    (define A (random-integer 10))
    (define B (random-integer 10))

    (pair A B)

    (equal? A B)
  )))
(define samples (repeat 500 take-sample))
(hist samples "A, B")

(scatter samples "A and B, conditioned on A = B")
~~~~

Now, of course, A and B go from being independent a priori to being perfectly correlated in the conditional distribution.  Try out these other conditioners to see other possible patterns of conditional dependence for a priori independent functions:

~~~~ {.norun}
(< (abs (- A B)) 2)

(and (>= (+ A B) 9) (<= (+ A B) 11))

(equal? 3 (abs (- A B)))

(equal? 3 (mod (- A B) 10))

(equal? (mod A 2) (mod B 2))

(equal? (mod A 5) (mod B 5))
~~~~

# Non-monotonic Reasoning

One reason explaining away is an important phenomenon in probabilistic inference is that it is an example of *non-monotonic* reasoning.  In formal logic, a theory is said to be monotonic if adding an assumption (or formula) to the theory never reduces the set of consequences of the previous assumptions.
Most traditional logics (e.g. First Order) are monotonic, but human reasoning does not seem to be. For instance, if I tell you that tweety is a bird, you conclude that he can fly; if I now tell you that tweety is an *ostrich* you retract the conclusion that he can fly. Over the years many non-monotonic logics have been introduced to model aspects of human reasoning. One of the first ways in which probabilistic reasoning with Bayesian networks was recognized as important for AI was that it could perspicuously capture these patterns of reasoning <ref>See for instance: Pearl, Probabilistic Reasoning in Intelligent Systems: Networks of Plausible Inference, 1988.</ref>.

Another way to think about monotonicity is by considering the trajectory of our belief in a specific proposition, as we gain additional relevant information.  In traditional logic, there are only three states of belief: true, false, and unknown (when neither a proposition nor its negation can be proven).  As we learn more about the world, maintaining logical consistency requires that our belief in any proposition only move from unknown to true or false. That is our "confidence" in any conclusion only increases (and only does so in one giant leap from unknown to true or false).

In a probabilistic approach, by contrast, there is a spectrum of degrees of belief.  We can think of confidence as a measure of how far our beliefs are from a uniform distribution, or how close to the extremes of 0 or 1.  In probabilistic inference, unlike in traditional logic, our confidence in a proposition can both increase and decrease.  As we will see in the next example, even fairly simple probabilistic models can induce complex explaining-away dynamics that lead our degree of belief in a proposition to reverse directions multiple times as the conditioning set expands.

# Example: Trait Attribution

Often we have to make inferences about different types of entities and their interactions, and a highly interactive set of relations between the entities leads to very challenging explaining away problems.  Inference is computationally difficult in these situations but the inferences come very naturally to people, suggesting these are important problems that our brains have specialized somewhat to solve (our perhaps that they have evolved general solutions to these tough inferences).

A familiar example comes from reasoning about the causes of students' patterns of success and failure in the classroom.  Imagine yourself in the position of an interested outside observer -- a parent, another teacher, a guidance counselor or college admissions officer -- in thinking about these conditional inferences.  If a student doesn't pass an exam, what can you say about why he failed?  Maybe he doesn't do his homework, maybe the exam was an unfair, or maybe he was just unlucky?

~~~~ {.mit-church}
(define samples
  (mh-query 1000 10

   (define exam-fair (flip .8))
   (define does-homework (flip .8))

   (define pass? (flip (if exam-fair
                           (if does-homework 0.9 0.6)
                           (if does-homework 0.4 0.2))))

   (list does-homework exam-fair)

   (not pass?)))

(hist samples "Joint: Student Does Homework?, Exam Fair?")
(hist (map first samples) "Student Does Homework")
(hist (map second samples) "Exam Fair")
~~~~

Now what if you have evidence from several students and several exams? We first re-write the above model to allow many students and exams:

~~~~ {.mit-church}
(define samples
  (mh-query 1000 10

   (define exam-fair-prior .8)
   (define does-homework-prior .8)
   (define exam-fair? (mem (lambda (exam) (flip exam-fair-prior))))
   (define does-homework? (mem (lambda (student) (flip does-homework-prior))))

   (define (pass? student exam) (flip (if (exam-fair? exam)
                                          (if (does-homework? student) 0.9 0.6)
                                          (if (does-homework? student) 0.4 0.2))))

   (list (does-homework? 'bill) (exam-fair? 'exam1))

   (not (pass? 'bill 'exam1))))

(hist samples "Joint: Bill Does His Homework?, Exam 1 Fair?")
(hist (map first samples) "Bill Does His Homework?")
(hist (map second samples) "Exam 1 Fair?")
~~~~

Initially we observe that Bill failed exam 1.  A priori, we assume that most students do their homework and most exams are fair, but given this one observation it becomes somewhat likely that either the student didn't study or the exam was unfair.

See how conditional inferences about Bill and exam 1 change as you add in more data about this student or this exam, or additional students and exams. Paste each of the data sets below into the above Church box, as a substitute for the current conditioner `(not (pass 'bill 'exam1))`.  Try to explain the dynamics of inference that result at each stage.  What does each new piece of the larger data set contribute to your intuition about Bill  and exam 1?

~~~~ {.norun}
  (and (not (pass? 'bill 'exam1)) (not (pass? 'bill 'exam2)))

  (and (not (pass? 'bill 'exam1))
       (not (pass? 'mary 'exam1))
       (not (pass? 'tim 'exam1)))

 (and (not (pass? 'bill 'exam1)) (not (pass? 'bill 'exam2))
       (not (pass? 'mary 'exam1))
       (not (pass? 'tim 'exam1)))

  (and (not (pass? 'bill 'exam1))
       (not (pass? 'mary 'exam1)) (pass? 'mary 'exam2) (pass? 'mary 'exam3) (pass? 'mary 'exam4) (pass? 'mary 'exam5)
       (not (pass? 'tim 'exam1)) (pass? 'tim 'exam2) (pass? 'tim 'exam3) (pass? 'tim 'exam4) (pass? 'tim 'exam5))

  (and (not (pass? 'bill 'exam1))
       (pass? 'mary 'exam1)
       (pass? 'tim 'exam1))

  (and (not (pass? 'bill 'exam1))
       (pass? 'mary 'exam1) (pass? 'mary 'exam2) (pass? 'mary 'exam3) (pass? 'mary 'exam4) (pass? 'mary 'exam5)
       (pass? 'tim 'exam1) (pass? 'tim 'exam2) (pass? 'tim 'exam3) (pass? 'tim 'exam4) (pass? 'tim 'exam5))

  (and (not (pass? 'bill 'exam1)) (not (pass? 'bill 'exam2))
       (pass? 'mary 'exam1) (pass? 'mary 'exam2) (pass? 'mary 'exam3) (pass? 'mary 'exam4) (pass? 'mary 'exam5)
       (pass? 'tim 'exam1) (pass? 'tim 'exam2) (pass? 'tim 'exam3) (pass? 'tim 'exam4) (pass? 'tim 'exam5))

  (and (not (pass? 'bill 'exam1)) (not (pass? 'bill 'exam2)) (pass? 'bill 'exam3) (pass? 'bill 'exam4) (pass? 'bill 'exam5)
       (not (pass? 'mary 'exam1)) (not (pass? 'mary 'exam2)) (not (pass? 'mary 'exam3)) (not (pass? 'mary 'exam4)) (not (pass? 'mary 'exam5))
       (not (pass? 'tim 'exam1)) (not (pass? 'tim 'exam2)) (not (pass? 'tim 'exam3)) (not (pass? 'tim 'exam4)) (not (pass? 'tim 'exam5)))
~~~~

# Example: Of Blickets and Blocking

A number of researchers have explored children's causal learning abilities by using the "blicket detector": a toy box that will light up when certain blocks, the blickets, are put on top of it. Children are shown a set of evidence and then asked which blocks are blickets. For instance, if block A makes the detector go off, it is probably a blicket. Ambiguous patterns are particularly interesting. Imagine that blocks A and B are put on the detector together, making the detector go off; it is likely that A is a blicket. Now B is put on the detector alone, making the detector go off; it is now less plausible that A is a blocket. This is called "backward blocking", and it is an example of explaining away.

We can capture this set up with a model in which each block has a persistent "blicket-ness" property, and the causal power of the block to make the machine go off depends on whether it is a blicket. Finally, the machine goes off if any of the blocks on it is a blicket (noisily).

~~~~
(define samples
  (mh-query 100 100

    (define blicket (mem (lambda (block) (flip 0.2))))
    (define (power block) (if (blicket block) 0.9 0.05))

    (define (machine blocks)
      (if (null? blocks)
          (flip 0.05)
          (or (flip (power (first blocks)))
              (machine (rest blocks)))))

    (blicket 'A)

    (machine (list 'A 'B))))

(hist samples "Is A a blicket?")
~~~~
Try the backward blocking scenario described above. Sobel, Tenenbaum, and Gopnik (2004) tried this with children, finding that four year-olds perform similarly to the model: evidence that B is a blicket explains away the evidence that A and B made the detector go away.

# A Case Study in Modularity: Visual Perception of Surface Lightness and Color

Visual perception is full of rich conditional inference phenomena, including both screening off and explaining away.
Some very impressive demonstrations have been constructed using the perception of surface structure by mid-level vision researchers; see the work of Dan Kersten, David Knill, Ted Adelson, Bart Anderson, Ken Nakayama, among others.
Most striking is when conditional inference appears to violate or alter the apparently "modular" structure of visual processing.  Neuroscientists have developed an understanding of the primate visual system in which processing for different aspects of visual stimuli -- color, shape, motion, stereo -- appears to be at least somewhat localized in different brain regions.  This view is consistent with findings by cognitive psychologists that at least in early vision, these different stimulus dimensions are not integrated but processed in a somewhat modular fashion.  Yet vision is at heart about constructing a unified and coherent percept of a three-dimensional scene from the patterns of light falling on our retinas.  That is, vision is causal inference on a grand scale.  Its output is a rich description of the objects, surface properties and relations in the world that are not themselves directly grasped by the brain but that are the true causes of the retinal stimulus.  Solving this problem requires integration of many appearance features across an image, and this results in the potential for massive effects of explaining away and screening off.

In vision, the luminance of a surface depends on two factors, the illumination of the surface (how much light is hitting it) and its reflectance. The actual luminance is the product of the two factors. Thus luminance is inherently ambiguous. The visual system has to determine what proportion of the luminance is due to reflectance and what proportion is due to the illumination of the scene. This has led to a famous illusion known as the *checker shadow illusion* discovered by Ted Adelson.

<img src='images/Checkershadow_illusion_small.png' width='400' />

The illusion results from the fact that in the image above both the square labeled A and the square labeled B are actually the same shade of gray. This can be seen in the figure below where they are connected by solid gray bars on either side.

<img src='images/Checkershadow_proof_small.png' width='400' />

What is happening here is that the presence of the cylinder is providing evidence that the illumination of square B is actually less than that of square A. Thus we perceive square B as having higher reflectance since its luminance is identical to square A and we believe there is less light hitting it. The following program implements a simple version of this scenario "before" we see the shadow cast by the cylinder.

~~~~ {.mit-church}
 (define noisy=
   (lambda (target value variance)
     (= 0 (gaussian (- target value) variance))))


 (define samples
   (mh-query
    100 100

    (define reflectance (gaussian 1 1))
    (define illumination (gaussian 3 0.5))
    (define luminance (* reflectance illumination))

    reflectance

    (noisy= 3.0 luminance 0.1)
   ))

 (truehist samples 10 "Reflectance")
~~~~

Here we have introduced a third kind of primitive random procedure, `gaussian` which outputs real numbers, in addition to `flip` (which outputs binary truth values) and `beta` (which outputs numbers in the interval [0,1]).  `gaussian` implements the well-known Gaussian or normal distribution. It takes two parameters: a mean, $\mu$, and a variance, $\sigma^2$.

<img src='images/Normal_distribution_pdf.png' width='400' />

The probability density function for the normal distribution is:
$$ P(x \mid \mu,\sigma) = \frac{1}{\sqrt{2\pi\sigma^2}}\; \exp{\Big[ \frac{-(x-\mu)^2}{2\sigma^2} \Big] } $$

Now let's condition on the presence of the cylinder.

~~~~ {.mit-church}
 (define noisy=
   (lambda (target value variance)
     (= 0 (gaussian (- target value) variance))))


 (define samples
   (mh-query
    100 100

    (define reflectance (gaussian 1 1))
    (define illumination (gaussian 3 0.5))
    (define luminance (* reflectance illumination))

    reflectance

    (and (noisy= 3.0 luminance 0.1) (noisy= 0.5 illumination 0.1))
   ))

 (truehist samples 10 "Reflectance")
~~~~

Conditional inference takes into account all the different paths through the generative process that could have generated the data. Two variables on different causal paths can thus become dependent once we condition on the way the data came out. The important point is that the variables `reflectance` and `illumination` are conditionally independent in the generative model, but after we condition on `luminance` they become dependent: changing one of them affects the probability of the other. This phenomenon has important consequences for cognitive science modeling.  Although our model of our knowledge of the world and language have a certain kind of modularity implied by conditional independence, as soon as we start using the model to do conditional inference on some data (e.g. parsing, or learning the language), formerly modularly isolated variables can become dependent.

### Other vision examples (to be developed)
Kersten's "colored Mach card" illusion is a beautiful example of both explaining away and screening off in visual surface perception, as well as a switch between these two patterns of inference conditioned on an auxiliary variable.

http://vision.psych.umn.edu/users/kersten/kersten-lab/Mutual_illumination/BlojKerstenHurlbertDemo99.pdf

Depending on how we perceive the geometry of a surface folded down the middle -- whether it is concave so that the two halves face each other or convex so that they face away -- the perceived colors of the faces will change as the visual system either discounts (explains away) or ignores (screens off) the effects of inter-reflections between the surfaces.

The two cylinders illusion of Kersten is another nice example of explaining away.  The gray shading patterns are identical in the left and right images, but on the left the shading is perceived as reflectance difference, while on the right (the "two cylinders") the same shading is perceived as due to shape variation on surfaces with uniform reflectance.

<!-- http://vision.psych.umn.edu/users/kersten/kersten-lab/images/twocylinders.gif -->

<img src='images/Kersten_et_al_explaining_away.png' width='400' />

(This image is from Kersten, Mamassian and Yuille, Annual Review of Psychology 2004)

<!-- model this using simple 1-dim procedural graphics:
   1-dim reflectance (assuming all changes are sharp discrete
   1-dim shape assuming all changes are gradual
   1-dim lighting assuming a gradient with some slope (1 param) falling off across the image, pos
       or neg slope
   Contour is described as either constant or sin function, and it is just a noisy diffusion of the
     3d shape map:

(define x-vals '(0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20))
(define contour-1 (map (lambda (x) 1) x-vals))
(define contour-2 (map (lambda (x) (sin (* (/ (modulo x 10) 10) 3.14159))) x-vals))

(define reflectance-changes (map (lambda (x) (- (sample-discrete '(1 20 1)) 1)) x-vals))
...

write the rendering function: image = illum * reflec * cos surface angle (check this
with http://en.wikipedia.org/wiki/Lambertian_reflectance)

compute simple discrete derivatives of shape for angle of surface normal

condition on effect of observing contour

  -->

# Exercises

1) Causal and statistical dependency. For each of the following programs:
	* Draw the dependency diagram (Bayes net). If you don't have software on your computer for doing this, Google Docs has a decent interface for creating drawings.
	* Use informal evaluation order reasoning and the intervention method to determine causal dependency between A and B.
	* Use conditioning to determine whether A and B are statistically dependent.

	A)
	
	~~~~ {data-exercise="ex1-1"}
	(define a (flip))
	(define b (flip))
	(define c (flip (if (and a b) 0.8 0.5)))
	~~~~
	
	B)
	
	~~~~ {data-exercise="ex1-2"}
	(define a (flip))
	(define b (flip (if a 0.9 0.2)))
	(define c (flip (if b 0.7 0.1)))
	~~~~
	
	C)
	
	~~~~ {data-exercise="ex1-3"}
	(define a (flip))
	(define b (flip (if a 0.9 0.2)))
	(define c (flip (if a 0.7 0.1)))
	~~~~
	
	D)
	
	~~~~ {data-exercise="ex1-4"}
	(define a (flip 0.6))
	(define c (flip 0.1))
	(define z (uniform-draw (list a c)))
	(define b (if z 'foo 'bar))
	~~~~
	
	E)
	
	~~~~  {data-exercise="ex1-5"}
	(define exam-fair-prior .8)
	(define does-homework-prior .8)
	(define exam-fair? (mem (lambda (exam) (flip exam-fair-prior))))
	(define does-homework? (mem (lambda (student) (flip does-homework-prior))))
	
	(define (pass? student exam) (flip (if (exam-fair? exam)
	                                       (if (does-homework? student) 0.9 0.5)
	                                       (if (does-homework? student) 0.2 0.1))))
	
	(define a (pass? 'alice 'history-exam))
	(define b (pass? 'bob 'history-exam))
	~~~~


#) Epidemiology: Imagine that you are an epidemiologist and you are determining people's cause of death. In this simplified world, there are two main diseases, cancer and the common cold. People rarely have cancer, $p( \text{cancer}) = 0.00001$, but when they do have cancer, it is often fatal, $p( \text{death} \mid \text{cancer} ) = 0.9$. People are much more likely to have a common cold, $p( \text{cold} ) = 0.2$, but it is rarely fatal, $p( \text{death} \mid \text{cold} ) = 0.00006$. Very rarely, people also die of other causes $p(\text{death} \mid \text{other}) = 0.000000001$.

	Write this model in Church program and use cosh to answer these questions (Be sure to include your code in your answer.):
	
	~~~~ {data-exercise="ex2"}
	;; use rejection-query and cosh for inference
	(rejection-query
	...)
	~~~~	
	
	A) Compute $p( \text{cancer} \mid \text{death} \wedge \text{cold} )$ and $p( \text{cancer} \mid \text{death} \wedge \text{no cold} )$. How do these probabilities compare to $p( \text{cancer} \mid \text{death} )$ and $p( \text{cancer} )$? Using these probabilities, give an example of explaining away.
	
	B) Compute $p( \text{cold} \mid \text{death} \wedge \text{cancer} )$ and $p( \text{cold} \mid \text{death} \wedge \text{no cancer} )$. How do these probabilities compare to $p( \text{cold} \mid \text{death} )$ and $p( \text{cold} )$? Using these probabilities, give an example of explaining away.
	
# References