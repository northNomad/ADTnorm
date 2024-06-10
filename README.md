
<!-- README.md is generated from README.Rmd. Please edit that file -->

# ADTGP

ADTGP uses Gaussian process regression to correct droplet-specific
technical noise in single-cell protein sequencing data

## Installation

You can install the development version of ADTGP from
[GitHub](https://github.com/) with:

``` r
# First install cmdstanr and stan
install.packages("cmdstanr", repos = c("https://mc-stan.org/r-packages/", getOption("repos")))
install_cmdstan()
set_cmdstan_path()

# install.packages("devtools")
devtools::install_github("northNomad/ADTGP")
```

## Example

Using ADTGP to obtain the conditional distribution of CD11b counts where
all cells share the same isotype control noise.

#### Step1: Prepare data and design matrix

``` r
library(ADTGP)

# Load dataset
data("idh_pdx")
d <- idh_pdx

## Relevel treatment
d$Treatment <- ifelse(d$Treatment == "veh", 1, 2)
```

#### Step2: Run ADTGP

``` r
## Fit model for CD11b
f_CD11b <- ADTGP(igg=d$IgG1,
                 poi=d$CD11b,
                 design_matrix=dm,
                 iter_warmup=3e3,
                 iter_sampling=1e3)
```

#### Trace plot

``` r
trace_cd11b <- f_CD11b$stanfit$draws(variables = "mu0")
trace_cd11b %>%
  as.data.frame() %>%
  mutate(iter=1:1000) %>%
  pivot_longer(1:4, names_to="chain", values_to="mu0") %>%
  ggplot(aes(x=iter, y=mu0,)) +
  geom_path(aes(color=chain)) +
  labs(x="MCMC Iteration", y=NULL) +
  scale_color_npg(name="Chain", labels=1:4) +
  theme_classic(15)
```

![](images/p_trace_cd11b.png)

#### Overlay prior and posterior distribution of CD11b counts

``` r
## Get prior distribution of count (50 draws)
m_prior <- ADTGP_prior(n=200L, igg=rep(0, 200))
index <- sample(1:4000, 50) 

## Downsample posterior distribution of count
posterior_counts <- f_CD11b$draws
posterior_counts <- posterior_counts[sample(1:nrow(posterior_counts), 200), ]

## Plot - CD11b expression prior and posterior
plot(NULL, xlim=c(0, 11), ylim=c(0, .2), xlab="log (n + 1)",
     ylab="Density", main="CD11b Expression")
for(i in index){
  lines(density(log(m_prior$draws[i, ] + 1)), col=col.alpha("black", .2))
}
lines(density(log(posterior_counts[, "T_rep_1"] + 1)), col="blue", lwd=2)
lines(density(log(posterior_counts[, "T_rep_2"] + 1)), col="firebrick", lwd=2)
```

![](images/density_cd11b_model.png)