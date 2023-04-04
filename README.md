# Control Vegas
This package is a wrapper over the [`vegas`](https://github.com/gplepage/vegas) integration package. The control variate variance reduction method is applied to the function when integrated along with the techniques applied in `vegas` such as importance sampling. To understand control variates, lets first look at how Monte carlo works.

## Monte Carlo Basics
For a continuous random variable $f(x)$, taking its expectation value where $x$ is drawn from some PDF $p(x)$ is
$$E_p[f]=\int f(x)p(x)\text{d}x$$
where the bounds on the integral are the limits of the PDF. Now, if we draw from the uniform distribution $U[a,b]$, then the PDF is $1/(b-a)$. So
$$E_{U[a,b]}[f]=\frac{1}{b-a}\int_a^bf(x)\text{d}x\qquad\text{or}\qquad(b-a)E[f]=\int_a^bf(x)\text{d}x$$
where the subscript on the expectation value is now assumed. Note that the left-hand side is just a normal integral. Namely, if we are looking to integrate some function $f(x)$ from $b$ to $a$, then that is equivalent to finding $(b-a)E[f]$. For $N$ discrete uniform random variables, the expectation value is just the mean:
$$E[X]=\frac{1}{N}\sum_{i=1}^NX_i$$
and this extends as an approximation for continuous random variables:
$$I=\int_a^bf(x)\text{d}x\approx\frac{b-a}{N}\sum_{i=1}^Nf(x_i)$$
where $x_i\sim U[a,b]$. Alternatively, if we consider the integral as sequentially summing up $N$ rectangles of equal length $(b-a)/N$ from $a$ to $b$, i.e. a Riemann sum, then we see that the $N$ rectangles gives an overall factor of $b-a$. The only difference between the two is one uses random variables and the breaks up the interval evenly and uses each point.

## Control Variate Basics
An equivalent integral to $I$ above is
$$I=\int_a^bf'(x)\text{d}x=\int_a^bf(x)-c\Big(g(x)-E[g]\Big)\text{d}x\approx\frac{b-a}{N}\sum_{i=1}^Nf(x_i)-c\Big(g(x_i)-E[g]\Big)$$
where $E[g]$ is the expectation value of $g(x)$ and is known. The constant $c$ can be chosen minimize the variance of $f'(x)$. First by finding $\text{Var}(f')$, taking the derivative with respect to $c$ and then setting it equal to zero, we find the optimal value
$$c^\*=-\frac{\text{Cov}(f,g)}{\text{Var}(g)}$$
and plugging in $c^*$ gives us a final variance of
$$\text{Var}(f')=\left(1-\frac{\text{Cov}(f,g)}{\text{Var}(f)\text{Var}(g)}\right)\text{Var}(f)=(1-\rho^2_{f,g})\text{Var}(f)$$
where $\rho_{f,g}$ is the correlation coefficient between $f$ and $g$. Since $|\rho_{f,g}|<1$, then this choice of $c$ will always decrease the variance. The same prescription can be applied for $n$ control variates, rather than just one which is used in this package.

A control variate is applied by using `vegas`'s importance sampling adaptation. Since $\rho$ is larger when the functions have a linear relationship, we use these previously adapted maps as the control variates. One can specify using the $i$th iteration of `vegas` as the control variate when initializing the `CVIntegrator` class or specify a list of iterations to use for multiple control variates.

## Installation
To install, `numpy` and `cython` must be installed beforehand due to the `gvar` package. Therefore, for `pip`:
```
python -m pip install numpy Cython && pip install control-vegas
```

## Usage
The workflow involes creating a `Function` class and then passing that to the `CVIntegrator`. The `Function` class contains the function to be integrated but also other information such as its name, the true value of the integration (if available) and parameters of the function. The `CVIntegrator` class does the integration and stores the results like mean and variance.

### Using a Built-In Function
For example,
```python
from control_vegas import CVIntegrator
from control_vegas.functions import NGauss

# Create 16-dimensional Gaussian
ng = NGauss(16)
# Print out all parameters of the class instance
print(ng, '\n')

# Create integrator class and use the 20th iteration as the control  variate
cvi = CVIntegrator(ng, evals=1000, tot_iters=50, cv_iters=20)
# Run the integration
cvi.integrate()

# Print info
cvi.compare(rounding=5)
```
This integrates a 16-dimensional Gaussian with 50 iterations and 5000 evaluations per iteration in `vegas`. And it uses the 20th iteration adapation from `vegas` as the control variate. The output is
```
NGauss(dimension=16, name=16D Gaussian, true_value=0.9935086032227194, mu=0.5, sigma=0.2) 

         |   No CVs    |  With CVs   
---------+-------------+-------------
Mean     |     0.99528 |     0.99579
Variance | 4.17808e-06 | 3.54189e-06
St Dev   |     0.00204 |     0.00188
VPR      |             |   15.22696%
```

### Adding Your Own Function
The `make_func` function allows you to make your own `Function` subclass. As an example, lets say $f(x_1,x_2)=ax_1^2+bx_2^2$ is a 2-dimensional function we want to integrate. Then we can do that by
```python
from control_vegas import CVIntegrator, make_func

# Create function, note that it is vetorized using Numpy slicing
def f(self, x):
    return self.a * x[:, 0]**2 + self.b * x[:, 1]

# Creating class with name 'WeightedPoly' and assigning values to the parameters in the function
wpoly = make_func('WeightedPoly', dimension=2, function=f, name='Weighted Polynomial', a=0.3, b=0.6)
# Print out parameters of class (note `true_value` isn't shown)
print(wpoly, '\n')

# Create integrator class and use the 20th iteration as the control  variate
cvi = CVIntegrator(wpoly, evals=1000, tot_iters=50, cv_iters=20)
# Run the integration
cvi.integrate()

# Print info
cvi.compare(rounding=5)
```
which outputs
```
WeightedPoly(dimension=2, name=WeightedPoly, a=0.3, b=0.6) 

         |     No CVs     |    With CVs    
---------+----------------+----------------
Mean     |     0.39996303 |     0.39993278
Variance | 5.27324871e-09 | 4.86145624e-09
St Dev   | 7.26171379e-05 | 6.97241439e-05
VPR      |                |    7.80908493%
```
The reason the function is vectorized is because, on the backend, `vegas`'s `batchintegrand` is used which can greatly speed up the computation.

### Manual Use of `Function` Class
To access the function call, use `function` or `f`. So, using the second example, I can run
```python
wpoly.f([1.2, 1.5]) # returns 1.3319999999999999
wpoly.f([0.2, 1], [0.8, 0.8], [1, 2]) # returns array([0.612, 0.672, 1.5  ])
```
This wraps around the private, vectorized `_function`/`_f` used by `vegas` so you don't have to worry about the proper `numpy` array shape

## Notes
- By default, functions are integrated from 0 to 1.
