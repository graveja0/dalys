---
title: Modeling Disability-Adjusted Life Years for Policy and Decision Analysis
subtitle: A Tutorial
keywords:
  - Cost-Effectiveness Analysis
  - Microsimulation
  - Discrete Event Simulation
  - Markov Cohort Models
abstract: |
  This study outlines a methodological framework for joint modeling of Disability- and Quality-Adjusted Life Year outcomes. Our primary focus is on how transition matrices and state occupancy payoffs in discrete-time Markov cohort models can be structured to calculate years of life lost to disability (YLD) and years of life lost to premature death (YLL), in addition to quality-adjusted life year (QALY) outcomes. We also demonstrate how our modeling framework extends directly to microsimulation and (in part) to continuous time discrete event simulation (DES) models. In a tutorial application, we use our joint modeling framework to construct a discrete time Markov cohort natural history model for cardiovascular disease that estimates DALY and QALY outcomes for any country, region, or setting represented in the 2020 Global Burden of Disease data.
plain-language-summary: |
  Structuring Markov Models for Multidimensional Health Outcomes
key-points:
  - Key Point 1
  - Key Point 2
date: last-modified
bibliography: references.bib
citation:
  container-title: Citation title
number-sections: true
editor_options: 
  chunk_output_type: console
prefer-html: true  
---


## Introduction {#sec-introduction}


::: {.cell}

```{.r .cell-code .hidden}
library(tidyverse)
```

::: {.cell-output .cell-output-stderr .hidden}

```
── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
✔ dplyr     1.1.4     ✔ readr     2.1.4
✔ forcats   1.0.0     ✔ stringr   1.5.1
✔ ggplot2   3.4.4     ✔ tibble    3.2.1
✔ lubridate 1.9.3     ✔ tidyr     1.3.0
✔ purrr     1.0.2     
── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
✖ dplyr::filter() masks stats::filter()
✖ dplyr::lag()    masks stats::lag()
ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors
```


:::

```{.r .cell-code .hidden}
library(MASS)
```

::: {.cell-output .cell-output-stderr .hidden}

```

Attaching package: 'MASS'

The following object is masked from 'package:dplyr':

    select
```


:::

```{.r .cell-code .hidden}
# library(Matrix)
library(expm)
```

::: {.cell-output .cell-output-stderr .hidden}

```
Loading required package: Matrix

Attaching package: 'Matrix'

The following objects are masked from 'package:tidyr':

    expand, pack, unpack


Attaching package: 'expm'

The following object is masked from 'package:Matrix':

    expm
```


:::

```{.r .cell-code .hidden}
library(knitr)
library(kableExtra)
```

::: {.cell-output .cell-output-stderr .hidden}

```

Attaching package: 'kableExtra'

The following object is masked from 'package:dplyr':

    group_rows
```


:::

```{.r .cell-code .hidden}
library(dampack)
library(here)
```

::: {.cell-output .cell-output-stderr .hidden}

```
here() starts at /Users/johngraves/Dropbox/Projects/dalys
```


:::

```{.r .cell-code .hidden}
library(Hmisc)
```

::: {.cell-output .cell-output-stderr .hidden}

```

Attaching package: 'Hmisc'

The following objects are masked from 'package:dplyr':

    src, summarize

The following objects are masked from 'package:base':

    format.pval, units
```


:::

```{.r .cell-code .hidden}
library(hrbrthemes)
```

::: {.cell-output .cell-output-stderr .hidden}

```
NOTE: Either Arial Narrow or Roboto Condensed fonts are required to use these themes.
      Please use hrbrthemes::import_roboto_condensed() to install Roboto Condensed and
      if Arial Narrow is not on your system, please see https://bit.ly/arialnarrow
```


:::

```{.r .cell-code .hidden}
library(ggsci)
options(scipen = 5) 
transpose <- purrr::transpose
select <- dplyr::select
#quarto preview index.qmd --to html --no-watch-inputs --no-browse
```
:::


Disability-adjusted life years (DALYs) measure disease burden in a population. Conceptualized in the Global Burden of Disease (GBD) study [@Murray1997], DALYs quantify the total sum of years of life lost due to disability attributable to a disease (YLD), plus years of life lost to premature mortality from the disease (YLL; i.e., DALY = YLD + YLL).

In addition to their role in describing levels and trends in disease burdens worldwide, DALYs are a primary health outcome in evaluations of health interventions in low- and middle-income countries (LMICs). In these settings, resource allocation decisions are guided by modeled assessments of the incremental costs per DALY averted under alternative (often competing) strategies to improve population health.[^1]

[^1]: The adoption of DALYs over other common health outcomes in health economics (e.g., quality-adjusted life years, or QALYs) stems from several practical and theoretical considerations. See @Feng2020 and @Wilkinson2016 for futher discussion.

Despite the prominent role of DALYs in global health policy, scant methodological guidance is available for adapting and/or structuring decision analytic models for DALY outcomes. This methodological gap has its roots in health economics education, where textbooks and training exercises focus almost exclusively on Quality-Adjusted Life Year (QALY) outcomes---the primary health outcome used for health technology assessments (HTAs) and policy decisionmaking in high-income countries (HICs). DALYs differ from QALYs in important and model-relevant respects, including the use of reference life tables to calculate YLLs and standardized disability weights to calculate YLDs.[^2] To the extent DALY-specific modeling considerations are taught, they are often considered in isolation and without a firm methodological grounding in *how* one might structure a model to measure DALY outcomes.

[^2]: In contrast, QALYs are calculated based on utility weights derived from general and patient surveys. See @Feng2020 and @Wilkinson2016 for futher discussion.

As a consequence, and in practice, health economic applications often resort to shortcuts and other "hacks" for calculating DALYs. For example, practitioners may simply estimate a "QALY-like" DALY that is based on a diseased state occupancy payoff of one minus the disability weight. Other approaches define a diseased-state payoff using the disability weight as an estimate of YLDs, and accumulate person-years in an absorbing death state (due to disease) as an estimate of YLLs. As this study will show, these shortcuts do not provide an accurate representation of DALY levels in a population.

This tutorial outlines a framework for direct incorporation of DALY outcomes in common decision modeling environments. Our primary focus is on discrete-time Markov cohort models---however, our framework extends directly to microsimulation and is also easily adapted for continuous time discrete event simulation (DES) models. As such, our study provides a comprehensive roadmap for incorporating DALY outcomes into common decision modeling frameworks.

To maintain consistency within the literature, our tutorial builds on an existing didactic disease progression model [@alarid2023introductory]. The underlying discrete time Markov cohort model is time homogeneous---that is, transition probabilities do not vary as a function of age/time in model. However, the methods and code provided are easily adapted for time inhomogenous models. Finally, recognizing the wide spectrum of experience and programming comfort level among those constructing DALY-based models, we provide replication materials for implementing our approach in both R and Microsoft Excel. 

## Background {#sec-background}

DALYs are calculated from two components. First, conditions are assigned disability weights ($D$) ranging from zero to one, with zero representing the absence of the condition and one representing the highest burden a condition can inflict, equivocal to death. Years lost to disability (YLD) is defined as the disability weight multiplied by the number of years a person lives with the condition ($L$):

$$
YLD(L) = D \cdot L
$$ {#eq-yld1} 

The impact of disease on mortality is quantified using years of life lost to disease (YLL), which is based on remaining life expectancy $Ex(a)$ at the age of premature death from the disease ($a$). 

$$
YLL(a)= Ex(a)
$$ {#eq-yll1} .

DALYs are the sum of the two components:

$$
DALY(L,a) = YLD(L) + YLL(a)
$$ {#eq-daly}

In the original GBD study, age-weighting and time discounting practices were applied to DALY calculations [@Murray1997]. These methods respectively weighted the burden of illness more during adulthood than early childhood and old age, and valued present health over future years of illness by discounting YLD and YLL measures by 3% per year. From 2010 onwards, both practices were discontinued to make the DALY a more descriptive measure [@who2013methods].

While the GBD no longer uses age and time discounting for the purposes of documenting disease burdens worldwide, the World Health Organization's Choosing Interventions that are Cost-Effective (WHO-CHOICE) program recommends consideration of time discounting of health outcomes [@murray2020; @bertram2021]. We therefore adopt the WHO-CHOICE recommendation and include continuous-time discounting in our DALY modeling approach.[^whodisc]

[^whodisc]: Practitioners who do not wish to discount DALY outcomes can simply set the annual discount rate $r$ to zero. In addition, our approach differentiates from common practice in the use of a continuous-time discount factor (i.e., $\frac{1}{r}(1-e^{rt})$), rather than a discrete-time discount factor (i.e., $1/(1+r)^t$). We do so to maintain consistency with the original GBD equations---though an approach based on discrete-time discounting could be used and will yield broadly similar results. 

For an annual discount rate $r$, and at age $a$, the equation for YLDs is,

$$
YLD(a,L) = D  \left ( \frac{1}{r}\left(1-e^{-r(L+a) }\right) \right ).
$$ {#eq-yld}

Similarly, YLLs are calculated as,

$$
YLL(a)= \frac{1}{r}\left(1-e^{-r Ex(a)}\right).
$$ {#eq-yll} 

### Reference Life Expectancy

While the creation of the DALY measure was an important step in global health research, it has received scrutiny due to its inherent assumptions and value judgements. For example, calculating YLLs requires the use of a reference life table that provides an estimate of ...

## Model Overview

This tutorial builds on an existing progressive disease model in which healthy individuals develop a disease with two health states ("Sick" and "Sicker") [@alarid2023introductory]. Individuals can also transition to an absorbing death state due to causes unrelated to the disease (i.e., "background" mortality), or due to disease-specific causes. In addition, the model structure is homogeneous (i.e., transition rates do not vary as a function of time). This is a simplification to distill model complexity down to only those components needed to demonstrate our DALY approach; our replication code is structured in such a way as to easily incorporate transition rates that are a function of age/time in the model. 

We consider outcomes under four strategies:

- A **Standard of Care** strategy based on the baseline model parameters.
- **Strategy A**,  which improves the quality of life among individuals with the disease, but does not affect disease progression.
- **Strategy B**, which reduces the rate of progression from Sick to Sicker by 40%.
- **Composite Strategy AB**, which jointly implements strategies A and B. 
 
A state transition diagram is shown in @fig-model1. In the figure, nodes are health states and edges depict possible transitions among them. Edge labels are defined in terms of transition intensities (rates).  Other key model parameters are summarized in TK....

![State transition diagram for progressive disease model](images/state-transition-diagram-1.svg){#fig-model1}


::: {.cell}

```{.r .cell-code .hidden}
library(tidyverse)
library(MASS)
library(expm)
library(knitr)
library(kableExtra)
options(scipen = 5) 
transpose <- purrr::transpose
select <- dplyr::select
options(knitr.kable.NA = '')

gen_wcc <- function (n_cycles, method = c("Simpson1/3", "half-cycle", "none")) 
{
    if (n_cycles <= 0) {
        stop("Number of cycles should be positive")
    }
    method <- match.arg(method)
    n_cycles <- as.integer(n_cycles)
    if (method == "Simpson1/3") {
        v_cycles <- seq(1, n_cycles + 1)
        v_wcc <- ((v_cycles%%2) == 0) * (2/3) + ((v_cycles%%2) != 
                                                     0) * (4/3)
        v_wcc[1] <- v_wcc[n_cycles + 1] <- 1/3
    }
    if (method == "half-cycle") {
        v_wcc <- rep(1, n_cycles + 1)
        v_wcc[1] <- v_wcc[n_cycles + 1] <- 0.5
    }
    if (method == "none") {
        v_wcc <- rep(1, n_cycles + 1)
    }
    return(v_wcc)
}
```
:::

::: {.cell}

```{.r .cell-code .hidden}
params_ <- list(
    # Treatment Strategies
    v_tx_names = c("SoC","A","B","AB"),      # treatment names
    n_tx = 4, # number of treatment strategies
    
    cycle_correction = "half-cycle",
    
    v_tr_names = c("H","S1","S2"), # transient health states
    v_ab_names = c("DOC","DS"), # absorbing health states
    n_states = 5, # total number of health states
    
    horizon = 200,    # model time horizon (in years)  
    r_v_disc_h  = 0.03,     # annual discount rate for health outcomes
    r_v_disc_c = 0.03,     # annual discount rate for cost outcomes
    Delta_t = 1,      # time step (1 = yearly, 1/12 = monthly, etc.)
    age0 = 25,         # age at baseline
    v_s0T = c(1,0,0,0,0), # initial state occupancy  
                      # c(1,0,0,0,0) means the modeled cohort starts off healthy
    
    r_HS1 = 0.15,   # disease onset rate
    r_S1H = 0.5,    # recovery rate
    r_S1S2 = 0.105,   # disease progression rate
    r_HD = 0.002,    # background mortality rate
    
    hr_S1 = 3.0,     # hazard rate of disease-related death from S1 state
    hr_S2 = 10.0,    # hazard rate of disease-related death from S1 state
    
    u_S1 = 0.75,       # Sick utility weight
    u_S2 = 0.5,        # Sicker utility weight
    u_D = 0,           # Death utility weight
    u_H = 1,           # Healthy utility weight
    
    dw_S1 = 0.25,      # Sick disability weight
    dw_S2 = 0.5,       # Sicker disability weight
    
    c_H = 2000,   # annual cost of healthy
    c_S1 = 4000,  # annual cost of S1
    c_S2 = 15000, # annual cost of S2
    c_D = 0, # annual cost of death

    c_trtA = 12000, # cost of treatment A
    u_trtA = 0.95, # utility weight for treatment A (S1 state)
    dw_trtA = 0.08,    # Disability weight for sick under treatment A
    
    c_trtB = 12000, # cost of treatment B
    hr_S1S2_trtB = 0.6, # reduction in rate of disease progression 
    
    hr_treat = 0.85,  # Hazard Ratio for Treatment Strategy
    hr_prevent = 0.9, # Hazard Ratio for Prevention Strategy

    df_ExR =  # Reference life table from GBD
          tibble::tribble(
              ~Age, ~Life.Expectancy,
              0L,       88.8718951,
              1L,      88.00051053,
              5L,      84.03008056,
              10L,      79.04633476,
              15L,       74.0665492,
              20L,      69.10756792,
              25L,      64.14930031,
              30L,       59.1962771,
              35L,      54.25261364,
              40L,      49.31739311,
              45L,      44.43332057,
              50L,      39.63473787,
              55L,      34.91488095,
              60L,      30.25343822,
              65L,      25.68089534,
              70L,      21.28820012,
              75L,      17.10351469,
              80L,      13.23872477,
              85L,      9.990181244,
              90L,      7.617724915,
              95L,      5.922359078
          )
)

params <- 
    with(params_,{
        modifyList(params_,list(
            v_names_states = c(v_tr_names, v_ab_names), # health state names
            omega = horizon/Delta_t,  # Total number of cycles
            r_v_disc_h_Delta_t = r_v_disc_h * Delta_t,  # Cycle discount rate: health outcomes
            r_v_disc_c_Delta_t = r_v_disc_c * Delta_t,  # Cycle discount rate: cost outcomes
            ages = (0:(horizon/Delta_t))*Delta_t + age0,  # Age in each cycle
             # Approximation function for reference life table life expectancies:
            f_ExR = function(x) pmax(0,unname(Hmisc::approxExtrap(df_ExR$Age, df_ExR$Life.Expectancy,xout = x)$y))
        ))
    })

params$ages_trace <- params$ages
params$ages <- params$ages[-length(params$ages)]

v_disc_h =  # Continuous time discounting
  exp(-params$r_v_disc_h_Delta_t  * 0:(params$omega))
# v_disc_h =  # Discrete time discounting
#   with(params,1 / (( 1 + (r_v_disc_h * Delta_t)) ^ (0 : omega)))
v_disc_c = 
  exp(-params$r_v_disc_c_Delta_t  * 0:(params$omega))
```
:::




## Transition Matrices

With the model parameterized, our next step is to define the matrices that govern health state transitions. The state transition diagram represented in @fig-model1 is not well-suited to calculate DALY outcomes, however. A primary reason is that the death transitions reflect transitions due to all causes of death.  To calculate YLLs, we need to separately track the timing and number of deaths *due to disease*. 

To accommodate this need and to accurately model DALY outcomes, several options are available:

1. Re-define the health states to include a separate cause-specific death state as depicted in @fig-modelDS.[^othcause] We can then draw on the resulting Markov trace and use changes in the number of cause-specific deaths in each cycle to calculate YLLs. 

2. Include a non-Markovian transition state for cause-specific deaths in the transition matrix. This approach will maintain the Markovian components captured in @fig-model1, but will allow us to add a column to the Markov trace that separately tracks the number of new deaths from the disease in each cycle. We can then apply transition state payoffs (based on remaining life expectancy at each age/cycle) to calculate YLL outcomes. 

3. Define a block matrix Markov chain with rewards for occupancy (YLDs) and disease-relatd death transitions (YLLs) by adapting the methods in @caswell2021a and @caswell2018. This approach draws on matrix calculus and solves for expected outcomes as well as higher order moments such as variance and skewness. 

[^othcause]: In this example, disease-specific death rates are goverened by a hazard ratio applied to the background mortality rate. Because we are operating on the rate scale, we can separate out disease-related deaths from other-cause mortality by simply taking a difference in the rates. Other applications for prevalent conditions with high death rates, however, may require us to construct a cause-deleted life table to obtain background mortality rates that net out deaths from the modeled disease.

Each of the above approaches facilitate the design and execution of a decision-analytic model that accomodates any number or types of outcomes (e.g., QALYs, DALYs, life-years, etc.). In practice, approaches (1) and (2) draw on similar transition matrices and payoff vectors, as we will show immediately below. Approach (3) is quite distinct, however, so we will demonstrate this method separately in Section TK.

### Approach 1: Cause-Specific Death State

Under this approach, we separate out deaths from disease vs. other causes by defining a separate health state for cause-specific mortality; an updated state transition diagram is shown in @fig-modelDS. 

![State transition diagram for progressive disease model with separate cause-specific death state](images/state-transition-diagram-2.svg){#fig-modelDS}

Transitions among health states are defined in terms of continuous rates ("intensities") and are captured within an intensity matrix $\mathbf{Q}$,

![Transition Intensity Matrix for Approach 1](images/Q_model2.png)

Cell values in row $i$, column $j$ of $\mathbf{Q}$ capture the (continuous time) transition rate from health state $i$ to health state $j$. $\mathbf{Q}$ has diagonal elements defined as the negative sum of the off-diagonal row values (i.e., the row sums of $\mathbf{Q}$ are zero). This ensures that the Markov model is "closed"---that is, the total cohort size neither grows nor shrinks over time.

We next embed the continuous transition intensity matrix into a discrete time transition probability matrix by taking the matrix exponential of $\mathbf{Q}$ for a defined time step (i.e., "cycle length") $\Delta t$:[^embed]

$$
\mathbf{P} = e^{\mathbf{Q}\Delta t}
$$ {#eq-embed}

This results in a transition probability matrix with the following probabilities defined:

![Transition Probability Matrix for Approach 1](images/P_model2.png){width=60%}

[^embed]: In Markov theory, $\mathbf{P}$ is called the "discrete skeleton" of the continuous model [@iosifescu1980]. The conversion formula used to calculate $\widetilde{\mathbf{P}}$ is the matrix analogue to the standard rate-to-probability formula commonly taught in health economics textbooks, i.e., $p = 1 - e^{r\Delta t}$, where $r$ is the rate and $\Delta t$ is the time step (i.e., "cycle length").

Embedding the transition probability matrix using the matrix exponential ensures that the resulting transition probabilities capture the underlying continuous time disease process. In particular, $\mathbf{P}$ will capture the probability of compound ("jumpover") transitions within a single cycle. 

For example, in the continuous time rate matrix $\mathbf{Q}$ above, there is a zero-valued rate defined for progressions from Healthy (H) to Disease-related death (DS), since individuals must first become ill before they can die from disease-related causes. However, after embedding, the matrix $\mathbf{P}$ has a non-zero cycle transition probability from Healthy (H) to Disease-related death (DS) (i.e., $\texttt{p\_HDS}$). This value captures the probability of a compound or "jumpover" transition from Healthy and through the Sick and/or Sicker state to death from disease-related causes within the same discrete time cycle; see @graves2021 for further discussion, and @iosifescu1980 for additional theory.[^comparison]

[^comparison]: Because we embed the transition probability matrix using matrix exponentiation, rather than through pairwise application of rate-to-probability formulas to each transition type, our results will differ from those in @alarid2023introductory---even though we use identical input parameters. Application of standard rate-to-probability formulas in health states with competing risks (i.e., the possibility of transitioning to more than one other health state in a given cycle) will ignore the possibility of compound transitions within a single cycle. Though not (yet) widely used in health economics, embedding the transition probability matrix using the matrix exponential is the technically correct way to construct a transition probability matrix from underlying transition rates.

### Approach 2: Non-Markovian Tracking States

This section will outline an approach similar to Approach 1, but that draws on a non-Markovian "transition" state that tracks the number of disease-related deaths in each cycle; these counts will be used later to match the age of the cohort at each cycle with a reference life table to calculate YLL outcomes. 

Under this approach, we maintain the overall structure as depicted in the original model (@fig-model1), but augment the transition probability matrix with non-Markovian components to facilitate accounting of disease-related deaths.[^tracking]  Approach 2 offers a more generalized method that allows practitioners to accurately account for costs and/or health payoffs (such as YLLs) that are defined by *transitions* among health states, rather than occupancy in a health state. 

@fig-transition shows a state transition diagram with the tracking state added. The tracking state (shown as red nodes)  simply records transitions as cohort members move from either diseased state to the absorbing death state due to causes related to the disease. 

![State Transition Diagram with Transition State (Red)](images/state-transition-diagram-3.svg){#fig-transition}

[^tracking]: Tracking states also allow for accurate bookeeping for other outcomes such as costs. For example, if developing the disease incurs a one-time diagnosis or treatment cost, the compound transitions implied by the embedded transition probability matrix indicate that some individuals will transiently enter (and then exit) the Sick state in a single cycle. When calculating costs, practitioners may want to include a tracking state for the Sick state to be sure to capture these one-time costs, which would be masked if cost payoffs are simply multiplied by state occupancy at the end of each cycle (e.g., costs for individuals with a sojourn through the Sick state in a single cycle would not be accounted for).

In general, tracking states can either count the total number of transitions that have occurred up to a given cycle (i.e., an "accumulator" state), or can track the total number of new transitions that occur within a single cycle (i.e., a "transition" state).[^augmented] To calculate YLL outcomes we will add a transition state that records the total number of new disease-related deaths in each cycle. 

[^augmented]: More generally, accumulator and transition states can be defined for any number of transition types, as they are useful for capturing one-time costs in the model, or for for calculating other decision-relevant outcomes such as the total number of people who developed the disease or died from the disease as secondary outcomes.

To implement Approach 2, we add a transition state row and column to the transition intensity matrix. This transition state, called $\texttt{trDS}$, is included in the augmented intensity matrix $\mathbf{Q}$ below:

![Transition intensity matrix with transition state added](images/Q_model1.png)

Two aspects of $\mathbf{Q}$ are worth highlighting. First, $\mathbf{Q}$ is divided into a Markovian submatrix and the non-Markovian tracking row and column. This division is made apparent using dotted vertical and horizontal lines. Critically, the Markovian submatrix remains closed---that is, the diagonal elements remain unchanged so that the row sums of the submatrix remain zero, even after the addition of the tracking column along the "edges" of $\mathbf{Q}$. This ensures that the Markovian submatrix can be used to calculate state occupancy for a closed cohort that neither gains nor loses cohort members over the modeled time horizon.

Second, two transition intensities---from the S1 (Sick) and S2 (Sicker) states to Death---appear in the tracking column. This ensures that $\texttt{trDeadDisease}$ will track all relevant transitions to death due to the disease. Because we are operating on the rate scale, we can net out non-disease related deaths as captured by the background mortality rate among healthy individuals (i.e., $\texttt{r\_HD}$). 

As above, we obtain the transition probability matrix by embedding $\mathbf{Q}$ into the discrete time step (@eq-embed). However, the resulting transition probability matrix treats $\texttt{trDS}$ as an absorbing state (i.e., individuals are retained in the $\texttt{trDS}$ with probability one). Using the terminology introduced above, this absorbing state could serve as an **accumulator** state that (in the constructed Markov trace) records the total number of people who have died from the disease up to any given cycle. This may be a decision-relevant health outcome to consider on its own; indeed, so long as the Markovian submatrix remains closed, there is no limit to the number of accumulator and/or transition states one might add along the "edges" of a model.[^transex]

[^transex]: To build on the example of compound "jumpover" transitions above, suppose an individual starts off healthy in a cycle, then rapidly transitions through the Sick and Sicker state and dies due to disease-related causes within the same cycle. If there is some treatment cost associated with being in the Sicker state, a traditional approach that applies cost payoffs to state occupancy at the (beginning) end of the cycle would miss treatment costs for this individual because they *transition* through the Sicker state, but never occupy it at the beginning or end of a cycle. Adding a non-Markovian transition state to the model facilitates more accurate bookkeeping because the transition state would pick up on this transition through the Sicker state. 

To change $\texttt{trDS}$ to a **transition** state, we simply replace the absorbing probability of one in the cell $[\texttt{trDS},\texttt{trDS}]$ with a zero.  This cell-level change is highlighted in grey in the bottom right cell of $\mathbf{P}$ below:

![Transition Probability Matrix for Approach 2](images/P_model1.png){width=60%}


::: {.cell}

```{.r .cell-code .hidden}
fn_r_HD <- function(age) {
  # Access r_HD from the parent frame where this function is called
  r_HD <- get("r_HD", envir = parent.frame())
  r_HD
}

fn_r_HS1 <- function(age) {
  # Access r_HD from the parent frame where this function is called
  r_HS1 <- get("r_HS1", envir = parent.frame())
  r_HS1
}

fn_r_S1H <- function(age) {
  # Access from the parent frame where this function is called
  r_S1H <- get("r_S1H", envir = parent.frame())
  r_S1H
}

fn_r_S1S2 <- function(age) {
  # Access  from the parent frame where this function is called
  r_S1S2 <- get("r_S1S2", envir = parent.frame())
  r_S1S2
}

params1 <- with(params,modifyList(params,list(
    # Natural History Transition Rate Matrix
    m_R = 
      ages %>% map(~({
        mR_SoC = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_S1S2(.x),fn_r_HD(.x),hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        mR_A = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_S1S2(.x),fn_r_HD(.x),hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        mR_B = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + hr_S1S2_trtB * fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S1S2_trtB *  fn_r_S1S2(.x),fn_r_HD(.x),hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        mR_AB = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + hr_S1S2_trtB *  fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S1S2_trtB * fn_r_S1S2(.x),fn_r_HD(.x),hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        array(c(as.vector(mR_SoC),
                as.vector(mR_A), 
                as.vector(mR_B),
                as.vector(mR_AB)), 
              dim = c(length(v_tr_names)+ length(v_ab_names),length(v_tr_names)+ length(v_ab_names),length(v_tx_names)),
          dimnames = list(c(v_tr_names,v_ab_names),c(v_tr_names,v_ab_names),v_tx_names)) %>% 
            apply(.,3,function(x) x, simplify=FALSE) 
        
      }))
    )))

params1 <- with(params1,modifyList(params1,list(
    m_P = m_R %>% transpose() %>% map(~({
      mR_ = .x
      mR_ %>% map(~({
              expm(.x * Delta_t)
         }))
      }))
)))

params2 <- with(params,modifyList(params,list(
    v_tr_names = c("H","S1","S2"), # transient health states
    v_ab_names = c("D","trDS"), # absorbing health states
    n_states = 5, # total number of health states
    v_names_states = c(c("H","S1","S2"), c("D","trDS"))
)))

params2 <- with(params2,modifyList(params2,list(
    # Natural History Transition Rate Matrix
    m_R = 
      ages %>% map(~({
        mR_SoC = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_S1S2(.x),hr_S1 * fn_r_HD(.x),hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S2 * fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        mR_A = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),fn_r_S1S2(.x),hr_S1 * fn_r_HD(.x) ,hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S2 * fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        mR_B = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + hr_S1S2_trtB * fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S1S2_trtB *  fn_r_S1S2(.x),hr_S1 * fn_r_HD(.x) ,hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S2 * fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        mR_AB = 
          matrix(c(
          -(fn_r_HD(.x)+fn_r_HS1(.x)), fn_r_HS1(.x), 0, fn_r_HD(.x), 0,
          fn_r_S1H(.x),-(fn_r_S1H(.x) + hr_S1S2_trtB *  fn_r_S1S2(.x) + fn_r_HD(.x) + hr_S1 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S1S2_trtB * fn_r_S1S2(.x),hr_S1 * fn_r_HD(.x),hr_S1 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,-(fn_r_HD(.x) + hr_S2 * fn_r_HD(.x) - fn_r_HD(.x)),hr_S2 * fn_r_HD(.x),hr_S2 * fn_r_HD(.x) - fn_r_HD(.x),
          0,0,0,0,0,          
          0,0,0,0,0),
          nrow = n_states, 
          ncol = n_states,
          byrow=TRUE, 
          dimnames = list(c(v_tr_names,v_ab_names),
                          c(v_tr_names,v_ab_names)
          ))
        
        array(c(as.vector(mR_SoC),
                as.vector(mR_A), 
                as.vector(mR_B),
                as.vector(mR_AB)), 
              dim = c(length(v_tr_names)+ length(v_ab_names),length(v_tr_names)+ length(v_ab_names),length(v_tx_names)),
          dimnames = list(c(v_tr_names,v_ab_names),c(v_tr_names,v_ab_names),v_tx_names)) %>% 
            apply(.,3,function(x) x, simplify=FALSE) 
        
      }))
    )))

params2 <- with(params2,modifyList(params2,list(
    m_P = m_R %>% transpose() %>% map(~({
      mR_ = .x
      mR_ %>% map(~({
              tmp_ <- expm(.x * Delta_t)
              tmp_[5,5] = 0
              tmp_
         }))
      }))
)))
```
:::


## Construct the Markov Trace

With $\mathbf{P}$ defined under either Approach 1 or Approach 2, we now have the necessary ingredients to construct a Markov trace. Define $\mathbf{s}_0$ as the initial state occupancy vector at time $t=0$. The vector $\mathbf{s}_0$ has size $k$, where $k$ is the total number of states captured in the $k \times k$ matrix $\mathbf{P}$ (including transition states, if using Approach 2). This vector summarizes the number or fraction of the population in each health state at baseline. Often, this vector will be set such that the entire cohort starts off healthy---though this need not always be the case.

For a time-homogenous model such as considered here, health state occupancy at cycle $t$ is is calculated as:

$$
\mathbf{s}'_t=\mathbf{s}'_0 \mathbf{P}^t
$$ {#eq-trace}

For a time-inhomongenous model in which transition probabilities change over time (e.g., death rates increase due to aging), we must construct separate transition probability matrices for each cycle (i.e., $\mathbf{P}(t)$). State occupancy at cycle $t$ is calculated by sequentially applying the transition matrices corresponding to each time step leading up to cycle $t$, i.e., 

$$
\mathbf{s}'_t=\mathbf{s}'_0 \mathbf{P}(1)\mathbf{P}(2)\dots\mathbf{P}(t)
$$ {#eq-trace-inhomo}

@tbl-trace1 shows the Markov trace for the first five cycles under Approach 1, while @tbl-trace2 shows the trace under Approach 2.  @tbl-trace1 also includes a new column (```deaths_disease```) that is calculated as the difference in state occupancy in the ```DS``` column between cycles. This extra step will be necessary later for calculating YLL outcomes. The trace shown under Approach 2 (@tbl-trace2), by comparison, automatically calculates new deaths through the inclusion of the transition state ```trDS```; the values under ```deaths_disease``` and ```trDS``` are identical, again highlighting that either approach can be used to calculate the number of disease-related deaths in each cycle.  


::: {.cell}

```{.r .cell-code .hidden}
trace1 <- 
    with(params1, {
        m_P %>% map( ~ ({
            P = .x
            occ <- v_s0T
            P %>% map(~({
              occ <<- occ %*% .x
            })) %>% 
            map(~(data.frame(.x))) %>% 
            bind_rows()
        }))
    })  %>% 
    map(~({
        tmp = .x[1,]
        tmp[1,] = params1$v_s0T
        tmp = rbind(tmp,.x)
    }))

trace2 <- 
    with(params2, {
        m_P %>% map( ~ ({
            P = .x
            occ <- v_s0T
            P %>% map(~({
              occ <<- occ %*% .x
            })) %>% 
            map(~(data.frame(.x))) %>% 
            bind_rows()
        }))
    })  %>% 
    map(~({
        tmp = .x[1,]
        tmp[1,] = params2$v_s0T
        tmp = rbind(tmp,.x)
    }))
```
:::

::: {#tbl-trace1 .cell tbl-cap='Markov Trace Under Approach 1'}

```{.r .cell-code .hidden}
trace1$SoC %>% head() %>% 
  data.frame() %>% 
  mutate(deaths_disease = c(0,diff(DS))) %>% 
  mutate(cycle = row_number()-1) %>% 
  select(cycle,everything()) %>% 
  kable(digits = c(0,rep(4,6))) 
```

::: {.cell-output-display}
`````{=html}
<table>
 <thead>
  <tr>
   <th style="text-align:right;"> cycle </th>
   <th style="text-align:right;"> H </th>
   <th style="text-align:right;"> S1 </th>
   <th style="text-align:right;"> S2 </th>
   <th style="text-align:right;"> DOC </th>
   <th style="text-align:right;"> DS </th>
   <th style="text-align:right;"> deaths_disease </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 1.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 0.8870 </td>
   <td style="text-align:right;"> 0.1046 </td>
   <td style="text-align:right;"> 0.0062 </td>
   <td style="text-align:right;"> 0.0020 </td>
   <td style="text-align:right;"> 0.0003 </td>
   <td style="text-align:right;"> 0.0003 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 0.8232 </td>
   <td style="text-align:right;"> 0.1521 </td>
   <td style="text-align:right;"> 0.0197 </td>
   <td style="text-align:right;"> 0.0040 </td>
   <td style="text-align:right;"> 0.0010 </td>
   <td style="text-align:right;"> 0.0008 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 0.7832 </td>
   <td style="text-align:right;"> 0.1723 </td>
   <td style="text-align:right;"> 0.0363 </td>
   <td style="text-align:right;"> 0.0060 </td>
   <td style="text-align:right;"> 0.0022 </td>
   <td style="text-align:right;"> 0.0012 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 0.7547 </td>
   <td style="text-align:right;"> 0.1796 </td>
   <td style="text-align:right;"> 0.0540 </td>
   <td style="text-align:right;"> 0.0080 </td>
   <td style="text-align:right;"> 0.0037 </td>
   <td style="text-align:right;"> 0.0015 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 0.7321 </td>
   <td style="text-align:right;"> 0.1808 </td>
   <td style="text-align:right;"> 0.0717 </td>
   <td style="text-align:right;"> 0.0099 </td>
   <td style="text-align:right;"> 0.0056 </td>
   <td style="text-align:right;"> 0.0019 </td>
  </tr>
</tbody>
</table>

`````
:::
:::

::: {#tbl-trace2 .cell tbl-cap='Markov Trace Under Approach 2'}

```{.r .cell-code .hidden}
#|
trace2$SoC %>% head() %>% data.frame() %>% 
  mutate(cycle = row_number()-1) %>% 
  select(cycle,everything()) %>% 
  kable(digits = c(0,rep(4,5))) 
```

::: {.cell-output-display}
`````{=html}
<table>
 <thead>
  <tr>
   <th style="text-align:right;"> cycle </th>
   <th style="text-align:right;"> H </th>
   <th style="text-align:right;"> S1 </th>
   <th style="text-align:right;"> S2 </th>
   <th style="text-align:right;"> D </th>
   <th style="text-align:right;"> trDS </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:right;"> 0 </td>
   <td style="text-align:right;"> 1.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
   <td style="text-align:right;"> 0.0000 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 0.8870 </td>
   <td style="text-align:right;"> 0.1046 </td>
   <td style="text-align:right;"> 0.0062 </td>
   <td style="text-align:right;"> 0.0023 </td>
   <td style="text-align:right;"> 0.0003 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 0.8232 </td>
   <td style="text-align:right;"> 0.1521 </td>
   <td style="text-align:right;"> 0.0197 </td>
   <td style="text-align:right;"> 0.0050 </td>
   <td style="text-align:right;"> 0.0008 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 3 </td>
   <td style="text-align:right;"> 0.7832 </td>
   <td style="text-align:right;"> 0.1723 </td>
   <td style="text-align:right;"> 0.0363 </td>
   <td style="text-align:right;"> 0.0082 </td>
   <td style="text-align:right;"> 0.0012 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 0.7547 </td>
   <td style="text-align:right;"> 0.1796 </td>
   <td style="text-align:right;"> 0.0540 </td>
   <td style="text-align:right;"> 0.0117 </td>
   <td style="text-align:right;"> 0.0015 </td>
  </tr>
  <tr>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 0.7321 </td>
   <td style="text-align:right;"> 0.1808 </td>
   <td style="text-align:right;"> 0.0717 </td>
   <td style="text-align:right;"> 0.0155 </td>
   <td style="text-align:right;"> 0.0019 </td>
  </tr>
</tbody>
</table>

`````
:::
:::


## Occupancy and Transition Rewards

With the Markov trace constructed, we next turn our attention to defining payoff vectors for YLD and YLL outcomes. 

### Years of Life Lived with Disability (YLD)

To calculate YLDs, we define a disability weight payoff vector $\mathbf{d}_{YLD}$,

![YLD Payoff Vector](images/d_yld.png){width="60%"}

where $\texttt{dwS1}$ and $\texttt{dwS2}$ are the disability weights for the Sick and Sicker states, respectively. In addition, $r_{\Delta_t}$ is the cycle discount rate, which is calculated as,

$$
r_{\Delta_t} = r \Delta_t
$$ {#eq-cycledisc} where $r$ is the annual discount rate and $\Delta_t$ is the time step. 

In the YLD payoff vector, the term $\frac{1}{r_{\Delta_t}}(1-e^{-r_{\Delta_t}})$ is included as a continuous-time discounting adjustment factor for the defined cycle length $\Delta_t$. This term is included to discount time *within* each cycle in order to maintain the continuous-time discounting approach used in the original GBD equations.[^conttime] 


[^conttime]: For example, if $r=0.03$ and $\Delta_t=1$ (i.e., annual cycle length), this adjustment factor will be $0.985 = \frac{1}{r_{\Delta_t}}(1-e^{-r_{\Delta_t}})$. If $\Delta_t=1/12$ (monthly cycle) the cycle discounting adjustment factor is 0.99875.

To fully discount outcomes, we still must discount all future outcome values back to baseline ($t=0$). For a time-homogeneous model, discounted years of life lost to disability (YLD) at cycle $t$ is given by

$$
YLD(t)=\mathbf{s}'_0 \mathbf{P}^t \mathbf{d}_{YLD}  \times{e^{-r_{\Delta_t} t}}
$$ {#eq-yldt}

For a time-inhomogenous model, YLDs are calculated as
$$
YLD(t)=\mathbf{s}'_0 \mathbf{P}(1)\mathbf{P}(2)\dots\mathbf{P}(t)  \mathbf{d}_{YLD}  \times{e^{-r_{\Delta_t} t}}
$$ {#eq-yldt-inhomo}

### Years of Life Lost to Disease (YLLs)

As noted in @sec-background and in @eq-yll, YLLs are based on the present value of remaining life expectancy among deaths that occur in each cycle $t$. Define $a(t)$ as the age of the cohort at cycle $t$, i.e., $a(t) = t \cdot \Delta t + a0$, where $a_0$ is the age of the cohort at $t=0$. Define $e(t)=e(a(t))$ as the present value of remaining life expectancy at cycle $t$. Following the GBD continuous time discounting approach, $e(a(t))$ is given by

$$
e(a(t)) = \frac{1}{r}\big (1 - e^{-rEx(a(t))} \big )
$$ {#eq-pvEx}

where $Ex(a)$ is the remaining life expectancy at age $a$. $Ex(a)$ is drawn from either an exogenous (reference) or an endogenous life table, depending on the objectives of the modeling exercise [@anand2019].

We next define a remaining life expectancy payoff vector at cycle $t$:

![YLL Payoff Vector](images/d_yll.png){width="60%"}


This payoff vector reflects discounting, but only in terms of the present value of remaining life expectancy *at time* $t$.

$$
YLL(t)=\mathbf{s}'_t \mathbf{e}(t)  \times{e^{-r\Delta_t t}} =\mathbf{s}'_0 \mathbf{P}^t \mathbf{e}(t)  \times{e^{-r\Delta_t t}}
$$ {#eq-yllt}

Total discounted YLLs at time $t=0$ is given by:

$$
YLL=\sum_{t=0}^{N-1} YLL(t) =\sum_{t=0}^{N-1}\left(\mathbf{s}'_0 \mathbf{P}^t \mathbf{e}(t)  \times{e^{-r\Delta_t t}} \right) 
$$ {#eq-yllcum}


::: {.cell}

```{.r .cell-code .hidden}
# QALYs
qaly_ = with(params1,(matrix(c(u_H,
              u_S1 ,
              u_S2,
              u_D,
              u_D) * Delta_t,
            dimnames = list(c(
                c(v_tr_names,v_ab_names)
            ), c("UW")))
))
qaly_ <- 
  with(params1,{
    v_tx_names %>% map(~({
        if (.x=="A" | .x=="AB") {
          tmp_ <- qaly_
          tmp_[2,1] = u_trtA * Delta_t
          tmp_
      } else qaly_
    }))
  })

QALYt <- trace1 %>% map2(.,qaly_, ~ ({
    tmp = as.matrix(.x) %*% .y
    tmp 
}))

QALY = QALYt %>% map(~sum(.x * v_disc_h * gen_wcc(params1$omega, method = params1$cycle_correction))) 

# YLD
yld_ = with(params1,(matrix(c(0,
              dw_S1 * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t)) ,
              dw_S2 * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t)),
              0,
              0),
            dimnames = list(c(
                c(v_tr_names,v_ab_names)
            ), c("DW")))
))
yld_ <- 
  with(params1,{
    v_tx_names %>% map(~({
        if (.x=="A" | .x=="AB") {
          tmp_ <- yld_
          tmp_[2,1] = dw_trtA * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t))
          tmp_
      } else yld_
    }))
  })

YLDt <- trace1 %>% map2(.,yld_, ~ ({
    tmp = as.matrix(.x) %*% .y
    tmp 
}))

YLD = YLDt %>% map(~sum(.x*  v_disc_h * gen_wcc(params1$omega, method = params1$cycle_correction)))

# YLL
new_deaths_from_disease <- 
    map(trace1,~({
        c(0,diff(.x[,"DS"]))
    })) 
   
remaining_life_expectancy <- 
    with(params1,(1/r_v_disc_h) * (1 - exp(-r_v_disc_h * f_ExR(ages_trace))))
    
YLLt <- 
    new_deaths_from_disease %>% map(~(.x * remaining_life_expectancy ))

YLL <- 
    YLLt %>% map(~(sum(.x * v_disc_h * gen_wcc(params1$omega,method = params1$cycle_correction))))

DALY <- 
    map2(YLL,YLD,~(.x + .y))

# Accumualted Time in Absorbing Death State
accdaly_ = with(params1,(matrix(c(0,
              dw_S1 * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t)) ,
              dw_S2 * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t)),
              0,
              1) * Delta_t,
            dimnames = list(c(
                c(v_tr_names,v_ab_names)
            ), c("UW")))
))
accdaly_ <- 
  with(params1,{
    v_tx_names %>% map(~({
        if (.x=="A" | .x=="AB") {
          tmp_ <- accdaly_
          tmp_[2,1] = dw_trtA * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t))
          tmp_
      } else accdaly_
    }))
  })

ACCDALYt <- trace1 %>% map2(.,accdaly_, ~ ({
    tmp = as.matrix(.x) %*% .y
    tmp 
}))

ACCDALY = ACCDALYt %>% map(~sum(.x * v_disc_h * gen_wcc(params1$omega, method = params1$cycle_correction))) 

qaly_daly_ = with(params1,(matrix(c(u_H,
              1-dw_S1 ,
              1-dw_S2,
              u_D,
              u_D) * Delta_t,
            dimnames = list(c(
                c(v_tr_names,v_ab_names)
            ), c("QALY_DALY_")))
))
qaly_daly_ <- 
  with(params1,{
    v_tx_names %>% map(~({
        if (.x=="A" | .x=="AB") {
          tmp_ <- qaly_daly_
          tmp_[2,1] = (1-dw_trtA) * Delta_t
          tmp_
      } else qaly_daly_
    }))
  })

QALY_DALYt <- trace1 %>% map2(.,qaly_daly_, ~ ({
    tmp = as.matrix(.x) %*% .y
    tmp 
}))

QALY_DALY = QALY_DALYt %>% map(~sum(.x * v_disc_h * gen_wcc(params1$omega, method = params1$cycle_correction))) 

# Costs
cost_ = with(params1,(matrix(c(c_H,
              c_S1 ,
              c_S2,
              c_D,
              c_D)*Delta_t,
            dimnames = list(c(
                c(v_tr_names,v_ab_names)
            ), c("COST")))
))
cost_ <-
  with(params1, {
    v_tx_names %>% map( ~ ({
      if (.x == "A") {
        tmp_ <- cost_
        tmp_["S1", 1] = (c_S1 + c_trtA)*Delta_t
        tmp_["S2", 1] = (c_S2 + c_trtA)*Delta_t
        tmp_
      } else if (.x == "B") {
        tmp_ <- cost_
        tmp_["S1", 1] = (c_S1 + c_trtB)*Delta_t
        tmp_["S2", 1] = (c_S2 + c_trtB)*Delta_t
        tmp_
      } else if (.x == "AB") {
        tmp_ <- cost_
        tmp_["S1", 1] = (c_S1 + c_trtA + c_trtB)*Delta_t
        tmp_["S2", 1] = (c_S2 + c_trtA + c_trtB)*Delta_t
        tmp_
      } else cost_
    }))
  }) %>% 
  set_names(params1$v_tx_names)

COSTt <- trace1 %>% map2(.,cost_, ~ ({
    tmp = as.matrix(.x) %*% .y
    tmp 
}))

COST = COSTt %>% map(~sum(.x * v_disc_c * gen_wcc(params1$omega, method = params1$cycle_correction))) 

result1 <- cbind(YLD, YLL, DALY, ACCDALY, QALY_DALY, QALY,COST) %>%
  as.data.frame() %>%
  mutate_all( ~ as.numeric(.))  %>%
  rownames_to_column(var = "strategy") %>%
  mutate(approach = "Markov Trace") %>% 
  dplyr::select(approach, strategy, everything()) 
```
:::


## Approach 3: Markov Chain With Rewards

Define

$$
\begin{aligned}
\tau &= \text{Number of transient (non-absorbing) states}\\
\alpha &= \text{Number of absorbing states}\\
\omega &= \text{Number of cycles} \\
s &= \text{Total number of states; }s=\tau\omega+\alpha \\
\mathbf{U}_{x} &= \text{Transition matrix for age }x, \text{for }x=1,\dots,\omega\\
\mathbf{D}_{j} &=\text{Age advancement matrix for stage }j, \text{for }j=1,\dots,\tau \\
\mathbf{M}_{i} &= \text{Mortality matrix for age class }x, \text{for } x = 1,\dots\omega \\
\mathbf{K} &= \text{vec-permutation matrix; parameters }\tau,\omega
\end{aligned}
$$
In the above notation, the matrix $\mathbf{U}_x$ captures transition
probabilities among transient (i.e., non-absorbing) health states, while
$\mathbf{M}_x$ captures transitions from transient health states to the
absorbing death states (non-disease mortality and disease-related mortality). Indexing
by age class $x$ indicates that separate matrices are defined for each
age in the Markov model.

To construct $\mathbf{U}_x$ and $\mathbf{M}_x$ we define transition rate
("intensity") matrices as in Approaches 1 and 2 above.[^defineQ] The overall
intensity matrix $\mathbf{Q_x}$ is given by

[^defineQ]: The only difference with the two approaches above is that the rows
    in these rate matrices correspond to the final state, while the
    columns correspond to the starting state; this is the opposite of
    the rate matrices defined above, where the rows corresponded to the
    starting health state and the columns to the ending health state in
    a given cycle.

$$
\mathbf{Q}_x=\left(\begin{array}{c|c}
\mathbf{V}_x & \mathbf{0} \\
\hline \mathbf{S}_x  & \mathbf{0}
\end{array}\right)
$$ where $\mathbf{V}_x$ is the rate matrix for the transitory (i.e.,
non-absorbing) states and $\mathbf{S}_x$ is the rate matrix for the
absorbing states. The diagonal elements of $\mathbf{Q}_x$ are the negative sum of
the non-diagonal column elements, thus making the column sums of
$\mathbf{Q}_x$ zero.

For the defined time step $\Delta_t$, the discrete time
transition probability matrix $\mathbf{P}_x$ is obtained by taking the
matrix exponential of the intensity matrix ($\mathbf{Q}_x$) multipled by
the time step ($\Delta_t$):

$$
\mathbf{P}_x =e^{\mathbf{Q}_x  \Delta t}
$$

We can then obtain $\mathbf{U}_x$ and $\mathbf{M}_x$ based on:

$$
\mathbf{P}_x =\left(\begin{array}{c|c}
\mathbf{U}_x  & \mathbf{0} \\
\hline \mathbf{M}_x  & \mathbf{0}
\end{array}\right)
$$

In addition, the matrix $\mathbf{D}_j$ defines age advancement in the
Markov chain. Using the example from @caswell2021a, if $\omega=3$ then

$$
\mathbf{D}_j=\left(\begin{array}{ccc}
0 & 0 & 0 \\
1 & 0 & 0 \\
0 & 1 & {[1]}
\end{array}\right) \quad j=1, \ldots, \tau
$$

In our implementation, we include the (optional) 1 value in the lower
right corner; this assumes that after the last specified age, the cohort
continues to experience transitions among health states according to the
transition probabilities defined for the last age class. If this value
is 0, the model will assume that everyone dies after the last cycle.


We next combine the transition matrices (for all age classes) together
into a series of block-structured matrices as follows:

$$
\mathbb{U}=\left(\begin{array}{c|c|c}
\mathbf{U}_1 & \cdots & \mathbf{0} \\
\hline & \ddots & \\
\hline \mathbf{0} & \cdots & \mathbf{U}_\omega
\end{array}\right)
$$

$$
\mathbb{D}=\left(\begin{array}{c|c|c}
\mathbf{D}_1 & \cdots & \mathbf{0} \\
\hline & \ddots & \\
\hline \mathbf{0} & \cdots & \mathbf{D}_\tau
\end{array}\right)
$$

$$
\tilde{\mathbf{U}}=\mathbf{K}^{\top} \mathbb{D} \mathbf{K} \mathbb{U} \quad \tau \omega \times \tau \omega
$$

where $\mathbf{K}$ is a permutation matrix known as the vec-permutation
matrix.[^vecperm] See @henderson1981 and Appendix B in @caswell2021a for
further information.

[^vecperm]: A function to construct a vec-permultation matrix is provided
    within the code snippet below.

We also define

$$
\tilde{\mathbf{M}}=\left(\begin{array}{lll}
\mathbf{M}_1 & \cdots & \mathbf{M}_\omega
\end{array}\right) \quad \alpha \times \tau \omega
$$

and

$$
\tilde{\mathbf{P}}=\left(\begin{array}{c|c}
\tilde{\mathbf{U}} & \mathbf{0}_{\tau \omega \times \alpha} \\
\hline \tilde{\mathbf{M}} & \mathbf{I}_{\alpha \times \alpha}
\end{array}\right) \quad(\tau \omega+\alpha) \times(\tau \omega+\alpha)
$$ where $\mathbf{I}$ is the identity matrix and $\mathbf{0}$ is a matrix of zeros. 

We now (nearly) have the components needed to calculate outcomes. A key
difference in the Healthy Longevity approach, relative to the approaches
above, is that we do not calculate outcomes separately in each cycle, and then sum them. Rather, the
method utilizes matrix calculus to solve for *expected outcomes* and
other moments in the outcome distribution (e.g., variance, skewness,
etc.).[^expectedoutcomes]

[^expectedoutcomes]: Again, for our purposes here we will focus on expected
    outcomes---though note that formulas for higher-order moments are
    provided in @caswell2021a and @caswell2018.

To calculate outcomes, we must next define "reward" matrices
$\mathbf{R}_m$, where $m$ indexes the moment of interest (e.g., expected
value, variance, etc.). The structure and values of $\mathbf{R}_m$ will
differ, however, depending on the outcome.

To facilitate how we define rewards (i.e., payoffs), we briefly classify
each of our outcomes (LE, YLL, YLD, DALYs, QALYs) into broad classes
corresponding to whether the payoff or "reward" applies to occupancy, or
to transitions, within or to a given health state:

| Outcome                                   | Reward Class                                                            |
|---------------------------|---------------------------------------------|
| Life Expectancy                           | Occupancy (1.0 for each alive health state)                             |
| Years of life lived with disability (YLD) | Occupancy (disability weight applied to time in CVD state)              |
| Yearls of life lost to disease (YLL)      | Transition (remaining life expectancy applied to CVD death transitions) |
| Disability-Adjusted Life Years (DALYs)    | Occupancy (YLD) + Transition (YLL)                                      |
| Quality-Adjusted Life Years               | Occupancy (utility weights applied to living health states)             |

: Classification of Health Outcomes

### Occupancy-Based Outcomes

To calculate occupancy-based outcomes, we start with a reward matrix
$\mathbf{H}$, which has dimensions $\tau \times \omega$ and is
structured as shown in @fig-H-le:

![Reward matrix $\mathbf{H}$](images/H-LE.png){#fig-H-le fig-align="center" width="50%"}

Cell values within this matrix can be set to one if we want to "reward"
that health state-age combination in our outcome measure, and zero
otherwise.[^rewards]

[^rewards]: This structure allows us, for example, to estimate outcomes for
    certain age ranges or other decision-relevant combinations of health
    state and age.

For occupancy-based outcomes, we define $\mathbf{H}$ such that each cell
receives a value of one.

![Reward matrix $\mathbf{H}$](images/H-LE2.png){#fig-H fig-align="center" width="50%"}

We use this matrix to define the reward vector $\mathbf{h}$:[^stackreward]

[^stackreward]: The $\text{vec}$ operator stacks the columns of an $m \times n$
    matrix into a $mn \times 1$ vector.

$$
\mathbf{h} = \text{vec } \mathbf{H}
$$ We also define $\neg \mathbf{h}$ as the complement of $\mathbf{h}$,
(i.e., values of 1.0 become 0, and vice versa).

Occupancy-based outcomes with partial rewards (e.g., YLDs, QALYs)
require an additional matrix $\mathbf{V}$, which has the same structure
as $\mathbf{H}$:

If $\mathbf{S}$ is the set of states that receive a reward (e.g., the
CVD state for both QALYs and YLDs), then a cycle spent in state $i$ at
age $j$ is defined by

$$
\mathbf{V}(i, j)= \begin{cases}E\left[v(i, j)\right] & \text { if }(i, j) \in \mathcal{S} \\ 0 & \text { otherwise }\end{cases}
$$ For YLD outcomes, $\mathbf{V}$ is defined as shown in @fig-V-yld,
while for QALY outcomes $\mathbf{V}$ is defined as shown in @fig-V-qaly
.

![Reward matrix for YLD Outcome](images/H-YLD.png){#fig-V-yld
fig-align="center" width="70%"}

![Reward matrix \mathbf{V} for QALY
Outcome](images/H-QALY.png){#fig-V-qaly fig-align="center" width="60%"}

We simliarly define an occupancy indicator vector $\mathbf{v}$ just as
we did $\mathbf{h}$:

$$
\mathbf{v}_{m}=\operatorname{vec} \mathbf{V}_{m}
$$

### Rewards for Partial Occupancy

Like many health economic applications, the Healthy Longevity approach
makes assumptions on partial occupancy in a health state.[^partial]
Specifically, the approach assumes that partial occupancy in a health
state receives half the reward---essentially, we draw on an assumption
that mid-cycle transitions occur at the half-way point.[^cycle]

[^partial]: For example, half-cycle corrections are often used---though there
    are other methdos (e.g., Simpson's rule) that are also drawn upon.

[^cycle]: This is similar to the standard "half-cycle" correction, however
    under this approach, a half-cycle correction occurs in each cycle,
    not in the first and last cycles. As we will see below, for rewards
    that are uniform across transient health states (e.g., calculating
    life expectancy involves a "payoff" of 1.0 for every living health
    state) this will yield identical answers as a Markov trace-based
    approach that adopts a half-cycle correction.

We combine this partial ocupancy assumption, along with the vectorized
reward matrices $\mathbf{h}$ and $\mathbf{v}$ to obtain the following:

$$
\begin{aligned}
\tilde{\mathbf{B}}_{1} & =\mathbf{h} \mathbf{v}_{1}^{\top}+\frac{1}{2}(\neg \mathbf{h})\left(\mathbf{v}_{1}^{\top}\right)+\frac{1}{2}\left(\mathbf{v}_{1}\right)\left(\neg \mathbf{h}^{\top}\right) \\
\end{aligned}
$$ and

$$
\tilde{\mathbf{C}}_{1}=\frac{1}{2} \mathbf{1}_{\alpha} \mathbf{v_1}^{\top}
$$

We combine $\tilde{\mathbf{B}}_{1}$ and $\tilde{\mathbf{C}}_{1}$ to
obtain:

$$
\tilde{\mathbf{R}}_{1}=\left(\begin{array}{c|c}
\tilde{\mathbf{B}}_{1} & \mathbf{0} \\
\hline \tilde{\mathbf{C}}_{1} & \mathbf{0}
\end{array}\right) 
$$ which has same block structure as the transition probability matrix
$\tilde{\mathbf{P}}$.

Expected outcomes are based on

$$
\begin{aligned}
& \tilde{\boldsymbol{\rho}}_{1}=\tilde{\mathbf{N}}^{\top} \mathbf{Z}\left(\tilde{\mathbf{P}} \circ \tilde{\mathbf{R}}_{1}\right)^{\top} \mathbf{1}_{s} 
\end{aligned}
$$ where $\tilde{\mathbf{N}}$ is the fundamental matrix

$$
\tilde{\mathbf{N}}=(\mathbf{I}-\tilde{\mathbf{U}})^{-1}
$$ and $\mathbf{Z}$ is

$$
\mathbf{Z}=\left(\mathbf{I}_{\tau \omega} \mid \mathbf{0}_{\tau \omega \times \alpha}\right)
$$


### Transition-Based Outcomes

For transition-based outcomes such as YLLs, we define the first moment of remaining life expectancy as the
vector $\tilde{\boldsymbol{\eta}}^{\top}$. This vector has dimensions
$\tau\omega \times 1$ and has the following basic structure:

$$
\tilde{\mathbf{\eta}}=\left(\begin{array}{c}
\eta_{11} \\
\vdots \\
\eta_{\tau 1} \\
\hline \vdots \\
\hline \eta_{1 \omega} \\
\vdots \\
\eta_{\tau \omega}
\end{array}\right)
$$ 
where $\eta_{i x}$ is remaining life expectancy for an individual in
health state $i$ at a given age $x$. In this structure, remaining life expectancy for each health state is grouped within age classes. 

Our choice for remaining life expectancy values $\eta_{i x}$ for YLL outcomes will depend on
the context and research question at hand [@anand2019]. Historically, the GBD
has utilized an *exogenous*, external life table 
based on the maximum potential life span among humans [@globalburdenofdiseasecollaborativenetwork2021]. @anand2019 discuss alternative contexts in which remaining life expectancy based on an *endogenous* life table or life expectancy model might be preferred. 

The distinction between *exogenous* and *endogenous* rewards for YLLs boils down to whether the remaining life expectancy value used originates from *outside* the population under study (i.e., mortality risks used to calculate remaining life expectancy at a given age are independent of the mortality risks of the population being assessed), or not. 

For YLLs based on exogenous life tables, such as the reference life table published by the GBD, we define $\tilde{\boldsymbol{\eta}}^{\top}$ based on the reference life table value at each age. For YLLs based on an endogenous life table, we could simply swap in external life table values from our country or region of interest, *or* use the model itself to estimate remaining life expectancy for a given age and initial health state. 

We next construct the reward matrices:

$$
\begin{aligned}
\tilde{\mathbf{B}}_{1} & =\left(\mathbf{0}_{\tau \omega \times \tau \omega}\right) \\
\tilde{\mathbf{C}}_{1} & =\left(\begin{array}{c}
\tilde{\boldsymbol{\eta}}_{1}^{\top} \\
\mathbf{0}_{1 \times \tau \omega}
\end{array}\right) .
\end{aligned}
$$

and

$$
\tilde{\mathbf{R}}_{1}=\left(\begin{array}{c|c}
\mathbf{0}_{\tau \omega \times \tau \omega} & \mathbf{0}_{\tau \omega \times 2} \\
\hline \tilde{\boldsymbol{\eta}}_{1}^{\top} & \mathbf{0}_{1 \times 2} \\
\mathbf{0}_{1 \times \tau \omega} & \mathbf{0}_{1 \times 2}
\end{array}\right)
$$


Echoing the approach to occupancy-based rewards above, expected outcomes are based on

$$
\begin{aligned}
& \tilde{\boldsymbol{\rho}}_{1}=\tilde{\mathbf{N}}^{\top} \mathbf{Z}\left(\tilde{\mathbf{P}} \circ \tilde{\mathbf{R}}_{1}\right)^{\top} \mathbf{1}_{s} 
\end{aligned}
$$ where $\circ$ denotes element-wise multiplication, $\tilde{\mathbf{N}}$ is the fundamental matrix

$$
\tilde{\mathbf{N}}=(\mathbf{I}-\tilde{\mathbf{U}})^{-1}
$$ and $\mathbf{Z}$ is

$$
\mathbf{Z}=\left(\mathbf{I}_{\tau \omega} \mid \mathbf{0}_{\tau \omega \times \alpha}\right)
$$

## Total Expected Outcomes

For both occupancy- and transition-based outcomes, total (across all ages) outcomes for each starting health state are
calculated as

$$
\boldsymbol{\rho}_{m}^{\text {stage }}(\operatorname{age} x)=\left(\mathbf{e}_{x}^{\top} \otimes \mathbf{I}_{\tau}\right) \tilde{\boldsymbol{\rho}}_{m} \quad \tau \times 1
$$ where $\otimes$ is the Kronecker operator.


Alternatively, we may wish to calculate outcomes separately under
different starting ages, and for a specified starting health state
(e.g., healthy). This is given by

$$
\boldsymbol{\rho}_{m}^{\text {age }}(\text { stage } i)=\left(\mathbf{I}_{\omega} \otimes \mathbf{e}_{i}^{\top}\right) \tilde{\boldsymbol{\rho}}_{m} \quad \omega \times 1,
$$



::: {.cell}

```{.r .cell-code .hidden}
params3_ <- with(params1,modifyList(params1,list(
    alpha = length(v_ab_names),
    tau = length(v_tr_names), 
    s = length(v_tr_names)*omega + length(v_ab_names) #total number of states;s=τω+α
)))
params3_ <- with(params3_,modifyList(params3_,list(
  m_R_t = m_R %>% map(~({
    tmp <- .x
    tmp %>% map(~(t(.x)))
  }))
)))

# not sure why this is needed, but otherwise the length gets set too long...
params3 <- with(params3_, modifyList(params3_, list(m_R_ = m_R_t %>% transpose())))
params3$m_R = params3$m_R_

params3 <- with(params3,modifyList(params3,list(
    m_V = m_R %>% map(~({
            R = .x
            R %>% map(~({
              m <- .x[v_tr_names,v_tr_names] 
            }))
            
        })),
     
    m_Q = m_R %>% map(~({
      R = .x 
      R %>% map(~({
                V = .x[v_tr_names,v_tr_names]
                S = .x[v_ab_names,v_tr_names]
                zero_ <- matrix(0, nrow = length(v_tr_names)+length(v_ab_names), ncol = length(v_ab_names))
                tmp <- cbind(rbind(V,S),zero_)
                dimnames(tmp) <- list(c(v_tr_names,v_ab_names),c(v_tr_names,v_ab_names))
                tmp
      }))
    }))    
)))

params3 <- with(params3,modifyList(params3,list(
    m_P3 = m_Q %>% map(~({
          Q = .x
          Q %>% map(~(expm(.x * Delta_t)))
    }))
)))
# For some reason, can't replace m_P directly, so resorting to this .. 
params3$m_P = params3$m_P3

params3 <- with(params3,modifyList(params3,list(
    m_U = m_P %>% map(~({
          P <- .x 
          P %>% map(~(.x[v_tr_names,v_tr_names]))
    })),
    m_M = m_P %>% map(~({
        P = .x
        P %>% map(~(.x[v_ab_names,v_tr_names]))
        
    }))
)))

params3 <- with(params3,modifyList(params3,list(
    D = {
        # Create diagonal age advancement matrix
        D <- matrix(0, omega, omega)
        vec <- rep(1, omega-1)
        D[row(D)-1 == col(D)] <- vec
        D[omega,omega] = 1
        D
    }
)))

vec <-  # a simple function to return the vec of an array
    function(x) {
        y <- c(x)
        return(y)
    }

vecperm <- 
    # vecperm
    # function to calculate the vec permutation matrix K of index m,n
    # let X be a m x n matrix, and X' the transpose of X
    # then K satisfies 
    # vec(X') = K*vec(X)
    function(m, n) {
        K <- matrix(0, m * n, m * n)
        a <- matrix(0, m, n)
        
        for (i in 1:m) {
            for (j in 1:n) {
                e <- a
                e[i, j] <- 1
                K <- K + kronecker(e, t(e))
            }
        }
        
        return(K)
    }

params3 <- with(params3,modifyList(params3,list(
    bbD_ = kronecker(diag(tau), D),
    bbU_ =  m_U %>% 
            map(~(bdiag(.x))),
    K = vecperm(tau, omega)
)))

params3 <- with(params3,modifyList(params3,list(
    mUtilde = bbU_ %>% map( ~ ({
        t(K) %*% bbD_ %*% K %*% .x
    }))
)))

params3 <- with(params3,modifyList(params3,list(
    mMtilde = m_M %>% map(~({
      do.call(cbind,.x) 
    }))  
)))

params3 <- with(params3,modifyList(params3,list(
    mPtilde =  map2(mUtilde, mMtilde,  ~ ({
            rbind(cbind(.x, matrix(0, tau * omega, alpha)) ,
                  cbind(.y, diag(alpha)))
        }))
)))

healthy_longevity_occupancy <- function(params, H, V) {
    with(params,{
        map(v_tx_names,~({
            U = mUtilde[[.x]]
            P = mPtilde[[.x]]
            v_ = V[[.x]]
            N = solve(diag(tau*omega)-U)
            h = vec(H) %>% as.matrix()
            not_h = 1-h
            v <- vec(v_) %>% as.matrix()
            B1 <- h %*% t(v) + 0.5 * (not_h %*% t(v)) + 0.5 * (v %*% t(not_h)) # Eq. 46
            C1 = 0.5 * (rep(1,alpha) %*%  t(v)) # Eq. 48
            R1 = rbind(cbind(B1, matrix(0, tau * omega, alpha)) ,
                              cbind(C1, diag(alpha))) 
            R2 = R1 * R1
            R3 = R1 * R1 * R1
            Z = cbind(diag(tau*omega),matrix(0,nrow=tau*omega, ncol=alpha))
            e = rep(1,s)
            rho1_ <- t(N)%*% Z %*% t(P * R1) %*% e
            # The following needs to be debugged
            # rho2_ <-
            #   N %*% (Z %*% t(.y * R1) %*% e + 2 * t(.x * B1) %*% rho1_)
            # B2 <- R2[1:(tau * omega), 1:(tau * omega)]
            # rho3_ <- t(N) %*% (Z %*% ((t(.y * R3) %*% e)) + 3 * (t(.x * B2) %*% rho1_) + 3 * (t(.x * B1) %*% rho2_))
            rho1_
        }))
    })
}

healthy_longevity_yll <- function(params, life_expectancy, disc) {
    with(params,{
        map2(mUtilde,mPtilde,~({
            U = .x
            P = .y
            N = solve(diag(tau*omega)-U)
            Z = cbind(diag(tau*omega),matrix(0,nrow=tau*omega, ncol=alpha))
            disc_ = rev(sort(rep(disc,length(v_tr_names))))
            eta1_ex_ = rev(sort(rep(life_expectancy,length(v_tr_names))))
            eta1_ex =  eta1_ex_
            
            B1 = matrix(0,nrow=tau*omega, ncol = tau*omega)
            C1 = rbind(matrix(0,nrow=1,ncol=tau*omega),eta1_ex*disc_) 
            R1 = cbind(rbind(B1,C1),matrix(0,nrow=tau*omega+2,ncol=2))
            R2 = R1 * R1
            R3 = R1 * R1 * R1
            Z = cbind(diag(tau*omega),matrix(0,nrow=tau*omega, ncol=alpha))
            e = rep(1,s)
            rho1_ = t(N) %*% Z %*% t(.y * R1) %*% e
            # The following needs to be debugged
            # rho2_ <-
            #   N %*% (Z %*% t(.y * R1) %*% e + 2 * t(.x * B1) %*% rho1_)
            # B2 <- R2[1:(tau * omega), 1:(tau * omega)]
            # rho3_ <- t(N) %*% (Z %*% ((t(.y * R3) %*% e)) + 3 * (t(.x * B2) %*% rho1_) + 3 * (t(.x * B1) %*% rho2_))
            rho1_
        }))
    })
}
```
:::

::: {.cell}

```{.r .cell-code .hidden}
H = with(params3,matrix(1,nrow=tau, ncol=omega))

with(params3,{
  V_LY <<- v_tx_names %>% map(~({
    v_ <- matrix(1,nrow=tau, ncol = omega) 
    v_
  })) %>% 
    set_names(v_tx_names)
})

with(params3,{
  V_YLD <<- v_tx_names %>% map(~({
    v_ <- matrix(0,nrow=tau, ncol = omega) 
    v_[2,] <- v_disc_h[-length(v_disc_h)]*dw_S1 * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t))
    v_[3,] <- v_disc_h[-length(v_disc_h)]*dw_S2 * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t))
    if (.x %in% c("A","AB")) {
      v_[2,] <- v_disc_h[-length(v_disc_h)]*dw_trtA * Delta_t * (1/r_v_disc_h_Delta_t) * (1 - exp(-r_v_disc_h_Delta_t))
    }
    v_
  })) %>% 
    set_names(v_tx_names)
})

with(params3,{
  V_QALY <<- v_tx_names %>% map(~({
    v_ <- matrix(0,nrow=tau, ncol = omega) 
    v_[1,] <- v_disc_h[-length(v_disc_h)]*u_H * Delta_t     
    v_[2,] <- v_disc_h[-length(v_disc_h)]*u_S1 * Delta_t 
    v_[3,] <- v_disc_h[-length(v_disc_h)]*u_S2 * Delta_t 
    if (.x %in% c("A","AB")) {
      v_[2,] <- v_disc_h[-length(v_disc_h)]*u_trtA * Delta_t 
    }
    v_
  })) %>% 
    set_names(v_tx_names)
})

with(params3,{
  V_COST <<- v_tx_names %>% map(~({
    v_ <- matrix(0,nrow=tau, ncol = omega) 
    v_[1,] <- v_disc_c[-length(v_disc_c)]*c_H * Delta_t     
    v_[2,] <- v_disc_c[-length(v_disc_c)]*c_S1 * Delta_t 
    v_[3,] <- v_disc_c[-length(v_disc_c)]*c_S2 * Delta_t 
    if (.x %in% c("A")) {
      v_[2,] <- v_disc_c[-length(v_disc_c)]*(c_S1 + c_trtA) * Delta_t 
      v_[3,] <- v_disc_c[-length(v_disc_c)]*(c_S2 + c_trtA) * Delta_t  
    }
    if (.x %in% c("B")) {
      v_[2,] <- v_disc_c[-length(v_disc_c)]*(c_S1 + c_trtB) * Delta_t 
      v_[3,] <- v_disc_c[-length(v_disc_c)]*(c_S2 + c_trtB) * Delta_t  
    }    
    if (.x %in% c("AB")) {
      v_[2,] <- v_disc_c[-length(v_disc_c)]*(c_S1 + c_trtA + c_trtB) * Delta_t 
      v_[3,] <- v_disc_c[-length(v_disc_c)]*(c_S2 + c_trtA + c_trtA) * Delta_t  
    }    
    v_
  })) %>% 
    set_names(v_tx_names)
})

LY3_ <- params3 %>% healthy_longevity_occupancy(H = H, V = V_LY)
QALY3_ <- params3 %>% healthy_longevity_occupancy(H = H, V = V_QALY)
YLD3_ <- params3 %>% healthy_longevity_occupancy(H = H, V = V_YLD)
COST3_ <- params3 %>% healthy_longevity_occupancy(H = H, V = V_COST)

remaining_life_expectancy <- with(params3,(1/r_v_disc_h) * (1 - exp(-r_v_disc_h * f_ExR(ages))))
YLL3_ <- params3 %>% healthy_longevity_yll(life_expectancy = remaining_life_expectancy, disc = v_disc_h[-length(v_disc_h)])
DALY3_ <- map2(YLL3_,YLD3_,~(.x+.y))

LY3 <- LY3_ %>% map(~({
  tmp <- (kronecker(t(c(1,rep(0,params3$omega-1))) ,diag(params3$tau)) %*% as.matrix(.x))
  tmp[1,1]
}))

QALY3 <- QALY3_ %>% map(~({
  tmp <- (kronecker(t(c(1,rep(0,params3$omega-1))) ,diag(params3$tau)) %*% as.matrix(.x))
  tmp[1,1]
}))

YLD3 <- YLD3_ %>% map(~({
  tmp <- (kronecker(t(c(1,rep(0,params3$omega-1))) ,diag(params3$tau)) %*% as.matrix(.x))
  tmp[1,1]
}))

YLL3 <- YLL3_ %>% map(~({
  tmp <- (kronecker(t(c(1,rep(0,params3$omega-1))) ,diag(params3$tau)) %*% as.matrix(.x))
  tmp[1,1]
}))

DALY3 <- DALY3_ %>% map(~({
  tmp <- (kronecker(t(c(1,rep(0,params3$omega-1))) ,diag(params3$tau)) %*% as.matrix(.x))
  tmp[1,1]
}))

COST3 <- COST3_ %>% map(~({
  tmp <- (kronecker(t(c(1,rep(0,params3$omega-1))) ,diag(params3$tau)) %*% as.matrix(.x))
  tmp[1,1]
}))

result3 <- cbind(YLD3, YLL3, DALY3, QALY3,COST3) %>%
  as.data.frame() %>%
  rename(YLD = YLD3, YLL = YLL3, DALY = DALY3, QALY = QALY3, COST = COST3) %>% 
  mutate_all( ~ as.numeric(.))  %>%
  rownames_to_column(var = "strategy") %>%
  mutate(approach = "Markov Chain With Rewards") %>% 
  dplyr::select(approach, strategy, everything())
```
:::


## Results


::: {.cell}

```{.r .cell-code .hidden}
result1 %>% 
  bind_rows(result3) %>% 
  select(-approach) %>% 
  kable(digits = 3, col.names = c("Scenario","YLDs","YLLs","DALYs","DALY-hack","QALY-like DALY","QALY","Costs"))  %>% 
  kable_styling() %>% 
  pack_rows("Approaches 1 and 2 (Markov Trace)", 1,4) %>% 
  pack_rows("Approach 3 (Markov Chain With Rewards)",5,8)
```

::: {.cell-output-display}
`````{=html}
<table class="table" style="margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> Scenario </th>
   <th style="text-align:right;"> YLDs </th>
   <th style="text-align:right;"> YLLs </th>
   <th style="text-align:right;"> DALYs </th>
   <th style="text-align:right;"> DALY-hack </th>
   <th style="text-align:right;"> QALY-like DALY </th>
   <th style="text-align:right;"> QALY </th>
   <th style="text-align:right;"> Costs </th>
  </tr>
 </thead>
<tbody>
  <tr grouplength="4"><td colspan="8" style="border-bottom: 1px solid;"><strong>Approaches 1 and 2 (Markov Trace)</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 4.607 </td>
   <td style="text-align:right;"> 2.683 </td>
   <td style="text-align:right;"> 7.290 </td>
   <td style="text-align:right;"> 9.558 </td>
   <td style="text-align:right;"> 21.870 </td>
   <td style="text-align:right;"> 21.870 </td>
   <td style="text-align:right;"> 158522.6 </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 4.006 </td>
   <td style="text-align:right;"> 2.683 </td>
   <td style="text-align:right;"> 6.689 </td>
   <td style="text-align:right;"> 8.957 </td>
   <td style="text-align:right;"> 22.480 </td>
   <td style="text-align:right;"> 22.588 </td>
   <td style="text-align:right;"> 292273.9 </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 3.817 </td>
   <td style="text-align:right;"> 2.028 </td>
   <td style="text-align:right;"> 5.845 </td>
   <td style="text-align:right;"> 7.679 </td>
   <td style="text-align:right;"> 23.695 </td>
   <td style="text-align:right;"> 23.695 </td>
   <td style="text-align:right;"> 255483.2 </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 3.081 </td>
   <td style="text-align:right;"> 2.028 </td>
   <td style="text-align:right;"> 5.109 </td>
   <td style="text-align:right;"> 6.943 </td>
   <td style="text-align:right;"> 24.442 </td>
   <td style="text-align:right;"> 24.574 </td>
   <td style="text-align:right;"> 374862.0 </td>
  </tr>
  <tr grouplength="4"><td colspan="8" style="border-bottom: 1px solid;"><strong>Approach 3 (Markov Chain With Rewards)</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 4.571 </td>
   <td style="text-align:right;"> 2.810 </td>
   <td style="text-align:right;"> 7.381 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> 22.312 </td>
   <td style="text-align:right;"> 158440.8 </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 3.972 </td>
   <td style="text-align:right;"> 2.810 </td>
   <td style="text-align:right;"> 6.782 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> 23.027 </td>
   <td style="text-align:right;"> 291264.5 </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 3.794 </td>
   <td style="text-align:right;"> 2.124 </td>
   <td style="text-align:right;"> 5.919 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> 24.150 </td>
   <td style="text-align:right;"> 255156.7 </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 3.060 </td>
   <td style="text-align:right;"> 2.124 </td>
   <td style="text-align:right;"> 5.184 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;"> 25.028 </td>
   <td style="text-align:right;"> 373906.6 </td>
  </tr>
</tbody>
</table>

`````
:::
:::

::: {.cell}

```{.r .cell-code .hidden}
dampack::calculate_icers(cost = result1$COST, effect = result1$QALY, strategies = result1$strategy) %>% 
  bind_rows(
    dampack::calculate_icers(cost = result3$COST, effect = result3$QALY, strategies = result3$strategy)
  )  %>% 
  bind_rows(
    dampack::calculate_icers(cost = result1$COST, effect = -result1$DALY, strategies = result1$strategy) %>% 
      mutate(Effect = -Effect)
  ) %>% 
  bind_rows(
    dampack::calculate_icers(cost = result3$COST, effect = -result3$DALY, strategies = result3$strategy) %>% 
      mutate(Effect = -Effect)
  )   %>% 
  bind_rows(
    dampack::calculate_icers(cost = result1$COST, effect = -result1$ACCDALY, strategies = result1$strategy) %>% 
            mutate(Effect = -Effect)
    
  ) %>%   
  bind_rows(
    dampack::calculate_icers(cost = result1$COST, effect = result1$QALY_DALY, strategies = result1$strategy)
  ) %>%  
  kable(digits = c(1,0,3,0,3,0,1)) %>% 
  kable_styling(bootstrap_options = c("striped", "condensed"),full_width = FALSE)  %>% 
  pack_rows("QALY - Approaches 1 & 2",1,4) %>% 
  pack_rows("QALY - Approach 3",5,8) %>% 
  pack_rows("DALY - Approaches 1 & 2",9,12) %>% 
  pack_rows("DALY - Approach 3",13,16) %>% 
  pack_rows("DALY-Shortcut",17,20) %>% 
  pack_rows("QALY-like DALY",21,24)
```

::: {.cell-output-display}
`````{=html}
<table class="table table-striped table-condensed" style="width: auto !important; margin-left: auto; margin-right: auto;">
 <thead>
  <tr>
   <th style="text-align:left;"> Strategy </th>
   <th style="text-align:right;"> Cost </th>
   <th style="text-align:right;"> Effect </th>
   <th style="text-align:right;"> Inc_Cost </th>
   <th style="text-align:right;"> Inc_Effect </th>
   <th style="text-align:right;"> ICER </th>
   <th style="text-align:left;"> Status </th>
  </tr>
 </thead>
<tbody>
  <tr grouplength="4"><td colspan="7" style="border-bottom: 1px solid;"><strong>QALY - Approaches 1 &amp; 2</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 158523 </td>
   <td style="text-align:right;"> 21.870 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 255483 </td>
   <td style="text-align:right;"> 23.695 </td>
   <td style="text-align:right;"> 96961 </td>
   <td style="text-align:right;"> 1.825 </td>
   <td style="text-align:right;"> 53142 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 374862 </td>
   <td style="text-align:right;"> 24.574 </td>
   <td style="text-align:right;"> 119379 </td>
   <td style="text-align:right;"> 0.879 </td>
   <td style="text-align:right;"> 135763 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 292274 </td>
   <td style="text-align:right;"> 22.588 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> D </td>
  </tr>
  <tr grouplength="4"><td colspan="7" style="border-bottom: 1px solid;"><strong>QALY - Approach 3</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 158441 </td>
   <td style="text-align:right;"> 22.312 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 255157 </td>
   <td style="text-align:right;"> 24.150 </td>
   <td style="text-align:right;"> 96716 </td>
   <td style="text-align:right;"> 1.839 </td>
   <td style="text-align:right;"> 52604 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 373907 </td>
   <td style="text-align:right;"> 25.028 </td>
   <td style="text-align:right;"> 118750 </td>
   <td style="text-align:right;"> 0.877 </td>
   <td style="text-align:right;"> 135383 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 291264 </td>
   <td style="text-align:right;"> 23.027 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> D </td>
  </tr>
  <tr grouplength="4"><td colspan="7" style="border-bottom: 1px solid;"><strong>DALY - Approaches 1 &amp; 2</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 158523 </td>
   <td style="text-align:right;"> 7.290 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 255483 </td>
   <td style="text-align:right;"> 5.845 </td>
   <td style="text-align:right;"> 96961 </td>
   <td style="text-align:right;"> 1.445 </td>
   <td style="text-align:right;"> 67111 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 374862 </td>
   <td style="text-align:right;"> 5.109 </td>
   <td style="text-align:right;"> 119379 </td>
   <td style="text-align:right;"> 0.736 </td>
   <td style="text-align:right;"> 162129 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 292274 </td>
   <td style="text-align:right;"> 6.689 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> D </td>
  </tr>
  <tr grouplength="4"><td colspan="7" style="border-bottom: 1px solid;"><strong>DALY - Approach 3</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 158441 </td>
   <td style="text-align:right;"> 7.381 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 255157 </td>
   <td style="text-align:right;"> 5.919 </td>
   <td style="text-align:right;"> 96716 </td>
   <td style="text-align:right;"> 1.463 </td>
   <td style="text-align:right;"> 66122 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 373907 </td>
   <td style="text-align:right;"> 5.184 </td>
   <td style="text-align:right;"> 118750 </td>
   <td style="text-align:right;"> 0.734 </td>
   <td style="text-align:right;"> 161675 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 291264 </td>
   <td style="text-align:right;"> 6.782 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> D </td>
  </tr>
  <tr grouplength="4"><td colspan="7" style="border-bottom: 1px solid;"><strong>DALY-Shortcut</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 158523 </td>
   <td style="text-align:right;"> 9.558 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 255483 </td>
   <td style="text-align:right;"> 7.679 </td>
   <td style="text-align:right;"> 96961 </td>
   <td style="text-align:right;"> 1.879 </td>
   <td style="text-align:right;"> 51608 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 374862 </td>
   <td style="text-align:right;"> 6.943 </td>
   <td style="text-align:right;"> 119379 </td>
   <td style="text-align:right;"> 0.736 </td>
   <td style="text-align:right;"> 162129 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 292274 </td>
   <td style="text-align:right;"> 8.957 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> D </td>
  </tr>
  <tr grouplength="4"><td colspan="7" style="border-bottom: 1px solid;"><strong>QALY-like DALY</strong></td></tr>
<tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> SoC </td>
   <td style="text-align:right;"> 158523 </td>
   <td style="text-align:right;"> 21.870 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> B </td>
   <td style="text-align:right;"> 255483 </td>
   <td style="text-align:right;"> 23.695 </td>
   <td style="text-align:right;"> 96961 </td>
   <td style="text-align:right;"> 1.825 </td>
   <td style="text-align:right;"> 53142 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> AB </td>
   <td style="text-align:right;"> 374862 </td>
   <td style="text-align:right;"> 24.442 </td>
   <td style="text-align:right;"> 119379 </td>
   <td style="text-align:right;"> 0.747 </td>
   <td style="text-align:right;"> 159721 </td>
   <td style="text-align:left;"> ND </td>
  </tr>
  <tr>
   <td style="text-align:left;padding-left: 2em;" indentlevel="1"> A </td>
   <td style="text-align:right;"> 292274 </td>
   <td style="text-align:right;"> 22.480 </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:right;">  </td>
   <td style="text-align:left;"> D </td>
  </tr>
</tbody>
</table>

`````
:::
:::

::: {.cell}

```{.r .cell-code .hidden}
dampack::calculate_icers(cost = result1$COST, effect = result1$QALY, strategies = result1$strategy) %>% 
  mutate(outcome = "QALY", approach = "Approach 1 & 2") %>% 
  bind_rows(
    dampack::calculate_icers(cost = result3$COST, effect = result3$QALY, strategies = result3$strategy) %>% 
       mutate(outcome = "QALY", approach = "Approach 3")
  )  %>% 
  bind_rows(
    dampack::calculate_icers(cost = result1$COST, effect = -result1$DALY, strategies = result1$strategy) %>% 
      mutate(Effect = -Effect) %>% 
      mutate(outcome = "DALY", approach = "Approach 1 & 2")
  ) %>% 
  bind_rows(
    dampack::calculate_icers(cost = result3$COST, effect = -result3$DALY, strategies = result3$strategy) %>% 
      mutate(Effect = -Effect) %>% 
      mutate(outcome = "DALY", approach = "Approach 3")
  )   %>% 
  bind_rows(
    dampack::calculate_icers(cost = result1$COST, effect = -result1$ACCDALY, strategies = result1$strategy) %>% 
    mutate(Effect = -Effect) %>% 
    mutate(outcome = "ACCDALY", approach = "Approach 1 & 2")  
  ) %>%   
  bind_rows(
    dampack::calculate_icers(cost = result1$COST, effect = result1$QALY_DALY, strategies = result1$strategy) %>% 
    mutate(outcome = "QALY-like DALY", approach = "Approach 1 & 2")      
  ) %>% 
  filter(!grepl("QALY",outcome)) %>% 
  ggplot(aes(x = Effect, y = Cost)) + geom_point(aes(colour = approach, shape = outcome),size=2) + 
  hrbrthemes::theme_ipsum() + ggsci::scale_color_aaas(name="Approach") + facet_wrap(~Strategy) + 
  scale_x_continuous(limits = c(0,10)) + scale_shape(name="Outcome") + theme(legend.position = "bottom")
```
:::



## Conclusion

## To Incorporate

- [Link](https://academic.oup.com/heapol/article/21/5/402/578296?login=false)
- [Link](https://link.springer.com/article/10.1007/s40258-022-00722-3)
## References {.unnumbered}

:::{#refs}

:::