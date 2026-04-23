<div class="meta-data">22 apr 2026 </div>

# A Gentle Introduction to Kalman Filters (2D Kalman Filter )

## Moving to Multiple Dimensions: Position and Velocity

In my [previous post](https://fade2black.github.io/blog/kalman1d/), we explored the foundational intuition behind the Kalman Filter using a **univariate** (single-state) approach. I showed how the filter balances a noisy measurement against a predicted state to find an optimal estimate. 1D example is great for grasping the core concepts of "prediction" and "update" operations.
However, real-world systems are rarely that simple.

In this post, instead of tracking a single value, I will track a **multivariate** state vector that describes a more realistic scenario: 1D motion where both position and velocity are linked.

## The Problem Statement
Imagine we are tracking an object moving along a straight line. We have a sensor that provides us with **position ($x$)**, but this sensor is noisy.
We do not have a sensor for **velocity ($v_x$)**, but we know that velocity directly influences where the object will be in the next time step.

Our goal is to use the relationship between these two variables to achieve two things:

1. Filter the noise out of our position measurements.
2. Infer the velocity, even though we never actually measure it.

In the next section I try to explain how the equations are derived, though not fully.
If you are not interested in math heavy part then you can directly jump to "Putting It All Together" section. 

## Filter Equations

### Prediction
We start the prediction step.

We treat our state variables as a Gaussian distribution.
In the univariate case our state consists of a single variable $x$ and we iteratively update
$\mu_x$ and $\sigma_x^2$. 

But now the position and velocity are not treated as independent Gaussians - 
the poistion depends on the velocity. 
The Kalman filter models them as a joint Gaussian distribution,
capturing both their individual uncertainties and their correlation. 
So now our state is a vector

$$ \begin{bmatrix} x \\ v_x \end{bmatrix} $$

and we iteratively predict and update the mean vector 

$$ \begin{bmatrix} \mu_x \\ \mu_{v_x} \end{bmatrix} $$

and the covariance matrix capturing both their individual uncertainties and their correlation

$$
\begin{bmatrix} \sigma_x^2 & \sigma_{x v_x} \\ \sigma_{v_x x} & \sigma_{v_x}^2 \end{bmatrix}
$$ 


So, as a Kalman Filter designer our goal is to derive the predictition and update equations. 


We start with the basic kinemtic equations which look like

$$ x = x_0 + vdt + \frac{1}{2}adt^2 $$

$$ v_x = v_0 + adt $$

As I mention in the previos post each step of the whole process consist of two operations: 

1. Predict: "a priori" estimate - our best guess before we have seen the latest sensor data, and
2. Update: "a posteriori" estimate - improved guess after we have seen the latest sensor data.


Let's discretize the time. Assume each time step is equal to some $\Delta t$ (say 0.1 sec). 
Then we could rewrite our equations as 

$$ x_k = x_{k-1} + v_{k-1}\Delta t + \frac{1}{2}a(\Delta t)^2 $$

$$ v_k = v_{k-1} + a\Delta t $$

In other words, if at the step $k-1$ the object has already moved $x_{k-1}$ meters,
then at the end of the $k$-th step the object will move $x_{k-1} + v_{k-1}\Delta t + \frac{1}{2}a(\Delta t)^2$ meters. If we set $x_0$ and $v_0$ to some initial values and if we know the value of $a$ at each
step then we could compute step by step all values for $x_k$ and $v_k$.

However, we have no idea about the true value of $a$ at each step. 
Since we do not know the true value of acceleration at each step, we model it as a random variable:

$$ a \sim \mathcal{N}(0, \sigma_a^2) $$

also known as "white noise".

Since the Kalman filtering is a "predict-then-update" game we need first find out how we predict (prediction model).
In other words, we are interested in deterministic equations predicting both location and the acceleration. 
One possible approach is simply choose the accelration to be its mean value, i.e. 0. Thus we have 

$$ x_k = x_{k-1} + v_{k-1}\Delta t + \frac{1}{2}\mathbb{E}[a](\Delta t)^2 = x_{k-1} + v_{k-1}\Delta t $$

$$ v_k = v_{k-1} + {E}[a]\Delta t = v_{k-1} $$

In a vector form

$$ \begin{bmatrix} x_k \\ v_k \end{bmatrix} = \begin{bmatrix} 1 & \Delta t \\ 0 & 1 \end{bmatrix} \begin{bmatrix} x_{k-1} \\ v_{k-1} \end{bmatrix}
$$

or

$$ \mathbf{x_k} = \mathbf{F}\mathbf{x_{k-1}} $$

where $\mathbf{F}$ is called a **transition matrix** (or **function**) and equal to

$$ \mathbf{F} = \begin{bmatrix} 1 & \Delta t \\ 0 & 1 \end{bmatrix} $$

This is actually our mean vector. 

Next we need to derive how we predict our uncertainties (variances and covariance).
I'll omit the heavy and tedious part of algebra and statistics and write down only the result
for the uncertainty equations.

$$ Var(x_k) = Var(x_{k−1}​) + Var(v_{k−1}​)(\Delta t)^2 + 2Cov(x_{k−1}​,v_{k−1​})\Delta t + \sigma_a(\Delta t )^4  $$

$$ Var(v_k) = Var(v_{k−1}​) + \sigma_a (\Delta t)^2 $$

$$ Cov(x_k, v_k) = Cov(x_{k-1}, v_{k-1}) + \Delta t Var(v_{k-1}) + \frac{1}{2}(\Delta t)^3 \sigma_a $$

To bring it to the matrix form let's introduce the **covariance matrix** $\mathbf{P}$

$$ 
\mathbf{P}_{k} = \begin{bmatrix} Var(x_k) &  Cov(x_k, v_k) \\  Cov(x_k, v_k) & Var(v_k) \end{bmatrix}
$$

and 

the **process noise matrix**

$$ 
\mathbf{Q} = \begin{bmatrix} (\Delta t)^4 & \frac{1}{2}(\Delta t)^3 \\ \frac{1}{2}(\Delta t)^3 & (\Delta t)^2  \end{bmatrix}\sigma_a
$$

then the matrix form of our uncertainty equations will look like the following


$$ \mathbf{P}_k = \mathbf{F}\mathbf{P}_{k-1}\mathbf{F}^T + \mathbf{Q} $$

Next is the update step.

### Update
The update step of the Kalman filter corrects the predicted state $\mathbf{x}_k$ 
estimate using new measurements $\mathbf{z}$. 
It refines the estimate of the system's state by combining the predicted state and the difference between
the predicted state and the measurement. The funny and "smart" name for this simple notion (difference) is **residual**.
The goal is to reduce the uncertainty using the latest measurement and predicted state values. In other words, the update step simply scales down the covariance matrix $\mathbf{P}_k$ which is our uncertatinty indicator.  

Let's start with the residual. From our problem statement we know that we can only read objects location. Therefore, our measurement is simply a postion scalar $z_k$. In scallar form the residual look like

$$ y_k = z_k - x_k $$

where $x_k$ is the latest predicted location. However, our state is a column vector $\mathbf{x_k}$ and 
hence in order to convert the residual in the matrix form we define a matrix $\mathbf{H} = [1, 0]$
and define the residual as 

$$ \mathbf{y}_k = [z_k] - [1, 0] \begin{bmatrix} x_k \\ v_k \end{bmatrix} = \mathbf{z}_k - \mathbf{H}\mathbf{x}_k $$ 

The matrix $\mathbf{H}$ projects $\mathbf{x}_k$ onto $x_k$ resulting in a $1\times 1$ matrix which is subtracted from $[z_k]$. So, we basically compute $z_k - x_k$ in a matrix form.


$$ [1, 0] \begin{bmatrix} Var(x_k) &  Cov(x_k, v_k) \\  Cov(x_k, v_k) & Var(v_k) \end{bmatrix} \begin{bmatrix} 1 \\ 0 \end{bmatrix} $$

$$ = [Var(x_k),  Cov(x_k, v_k ] \begin{bmatrix} 1 \\ 0 \end{bmatrix} = [Var(x_k)] $$

So, 

$$ \mathbf{S} = \mathbf{H}\mathbf{P}_k\mathbf{H}^T + \mathbf{R} = [Var(x_k)] + [\sigma_R] = [Var(x_k) + \sigma_R]$$ 

and the Kalman Gain is

$$ \mathbf{K} = \mathbf{P}_k\mathbf{H}^T \mathbf{S}^{-1} = \begin{bmatrix} \frac{Var(x_k)}{Var(x_k) + \sigma_R)} \\ \frac{Cov(x_k, v_k)}{Var(x_k) + \sigma_R)} \end{bmatrix} $$ 

Once we have the residual and the Kalman gain we can update the state 

$$ \mathbf{x_k} = \mathbf{x_k} + \mathbf{K}\mathbf{y_k} $$

and the covariance matrix is updated like

$$ \mathbf{P_k} = (\mathbf{I}− \mathbf{K}\mathbf{H})\mathbf{P_k} $$

## Putting It All Together

We are given an object moving across the $x$ axis with a variable velocity. 
We are able to measure only the location of the object and we have no idea about the accleration of the object.
However we do know the degree of uncertainty of the accleration in terms of variance $\sigma_a^2$ with $\mu_a = 0$, known as white noise.

We also know the measurement noise $\sigma_R^2$ - how much we trust in measurements. 
We recieve measurements every $\Delta t$ time units (e.g. seconds).


So, initially we have:

- $\Delta t$ - time step (e.g. $0.5$)
- $\sigma_a^2$ - variance of accleration (where $a$ is Gaussian with $\mu_a = 0$), process noise
- $\sigma_z^2$ - measurement noise


Then using those qunatities we set the transition matrix (function) as 

$$ \mathbf{F} = \begin{bmatrix} 1 & \Delta t \\ 0 & 1 \end{bmatrix} $$

the process noise matrix as 

$$ 
\mathbf{Q} = \begin{bmatrix} (\Delta t)^4 & \frac{1}{2}(\Delta t)^3 \\ \frac{1}{2}(\Delta t)^3 & (\Delta t)^2  \end{bmatrix}\sigma_a
$$

the measurement noise matrix as 

$$ \mathbf{R} = [\sigma_z^2] $$

the process noise matrix as 

$$ \mathbf{Q} = [\sigma_a^2] $$

and since we measure only the location (and no velocity) we set the projection vector $\mathbf{H}$ to

$$ \mathbf{H} = [1, 0]$$

Once we setup all necessary matrixes and vectors we iteratively perform prediction and update as following

**Initially**

1. $\mathbf{x}_0 = \text{some reasonable vector}$
2. $\mathbf{P}_0 = \text{some reasonable matrix}$

**For $k=1,2,\dots$ and measurements $\mathbf{z_k}$ repeat the following steps:**

**Predict:**

1. $\mathbf{x}_k = \mathbf{x}_{k-1}\mathbf{F}$
2. $\mathbf{P}_k = \mathbf{F}\mathbf{P}_{k-1}\mathbf{F}^T + \mathbf{Q}$

**Update:**

1. $\mathbf{S} = \mathbf{H}\mathbf{P}_k\mathbf{H}^T + \mathbf{R}$
2. $\mathbf{K} = \mathbf{P}_k\mathbf{H}^T\mathbf{S}^{−1}$ (Kalman gain)
3. $\mathbf{y} = \mathbf{z}_k − \mathbf{H}\mathbf{x}_k$ (Residual)
4. $\mathbf{x}_k = \mathbf{x}_k + \mathbf{K}\mathbf{y}$ (Update the state)
5. $\mathbf{P}_k = (\mathbf{I}− \mathbf{K}\mathbf{H})\mathbf{P}_k$ (Update the covariance matrix)


## Rust Implementation

```rust
use nalgebra::{Matrix1, Matrix2, RowVector2, Vector2};

type KVector = Vector2<f64>;
type KMatrix = Matrix2<f64>;
type KRowVector = RowVector2<f64>;

pub fn run(true_data: &Vec<KVector>, noisy_data: &Vec<KVector>, dt: f64, var_a: f64, var_z: f64) -> Vec<KVector>{
  // State vector: [position, velocity]
  let mut x = KVector::new(noisy_data[0].x, noisy_data[0].y);
  let f = KMatrix::new(1.0, dt, 0.0, 1.0);
  let mut p = KMatrix::new(500.0, 0.0, 0.0, 49.0);
  let q = build_process_noise_matrix(dt, var_a);
  let h = KRowVector::new(1.0, 0.0);
  let r = Matrix1::new(var_z);
 
  let mut filtered_data = Vec::new();

  for i in 0..true_data.len() {
    let z = Matrix1::new(noisy_data[i].x);
    let (x_pred, p_pred) = predict(&x, &f, &p, &q);
    let (x_upd, p_upd) = update(&x_pred, &h, &r, &p_pred, &z);
    
    filtered_data.push(x_upd);

    p = p_upd;
    x = x_upd;
  }

  filtered_data
}

fn predict(x: &KVector, f: &KMatrix, p: &KMatrix, q: &KMatrix) -> (KVector, KMatrix) {
  let x_pred = f * x;
  let p_pred = f * p * f.transpose() + q;

  (x_pred, p_pred)
}

fn update(x: &KVector, h: &KRowVector, r: &Matrix1<f64>, p_pred: &KMatrix, z: &Matrix1<f64>) -> (KVector, KMatrix) {
  let y = z - h*x;
  let s = h * p_pred * h.transpose() + r;
  let k = p_pred * h.transpose() * s.try_inverse().unwrap();
  let x_upd = x + k * y;
  let p_upd = (KMatrix::identity() - k * h) * p_pred;

  (x_upd, p_upd)
}

fn build_process_noise_matrix(dt: f64, var_a: f64) -> KMatrix {
  KMatrix::new(
    0.25 * dt.powi(4) * var_a,
    0.5 * dt.powi(3) * var_a,
    0.5 * dt.powi(3) * var_a,
    dt.powi(2) * var_a
  )
}
```

## Simulation

Consider a target moving along a horizontal straight path at distance of 750 meters. 
Assume that another object has sensors that receive the target's $x$-coordinate with a variance of 100, 
meaning the error deviates by ±10 meters. The target's initiali $x$ (location) is 10.
For simplicity, let’s assume the scenario takes place in a 2D $xy$ plane. 
I wrote (in C++) a simulation of the object "chasing" the target with the following parameters:"

- $\sigma_a = 0.0015$
- $\sigma_z = 100$
- $\Delta t = 0.05$

Initially the state vector is set to 

$$ \mathbf{x} = \begin{bmatrix} 200 \\ 200 \end{bmatrix} $$

Transition matrix is 

$$ \mathbf{F} = \begin{bmatrix} 1 & 0.05 \\ 0 & 1 \end{bmatrix} $$

and the measurement (projection) vector is 

$$ \mathbf{H} = [1, 0] $$


For simplicity I have used the Pure Pursuit Method where I always point directly at where the target is right now. 
For moving targets, this results in a curved "tail-chase" path. The Proportional Navigation is next time.😉

<video width="640" height="360" controls>
  <source src="../videos/sim1.mov">
  Your browser does not support the video tag.
</video>

The following logs show the Kalman filter in action.
```bash
1. Actual(x): 10.0288, Noisy(x): 6.1229, Filtered(x): 39.7647
2. Actual(x): 10.0575, Noisy(x): 13.5440, Filtered(x): 32.1846
3. Actual(x): 10.0863, Noisy(x): 3.1338, Filtered(x): 28.5397
4. Actual(x): 10.1151, Noisy(x): 8.4791, Filtered(x): 29.7435
5. Actual(x): 10.1440, Noisy(x): 18.1567, Filtered(x): 33.8192
6. Actual(x): 10.1729, Noisy(x): 5.3147, Filtered(x): 35.6195
7. Actual(x): 10.2018, Noisy(x): 16.5388, Filtered(x): 39.4727
...
196. Actual(x): 15.6898, Noisy(x): 3.8002, Filtered(x): 17.9968
197. Actual(x): 15.7188, Noisy(x): 9.4194, Filtered(x): 17.8752
198. Actual(x): 15.7479, Noisy(x): 27.4758, Filtered(x): 18.1168
199. Actual(x): 15.7770, Noisy(x): 20.3383, Filtered(x): 18.2120
200. Actual(x): 15.8061, Noisy(x): 21.8255, Filtered(x): 18.3349
...
2803. Actual(x): 96.9489, Noisy(x): 110.1581, Filtered(x): 96.9406
2804. Actual(x): 96.9809, Noisy(x): 103.3817, Filtered(x): 96.9955
2805. Actual(x): 97.0128, Noisy(x): 112.6890, Filtered(x): 97.0830
2806. Actual(x): 97.0447, Noisy(x): 102.3809, Filtered(x): 97.1340
2807. Actual(x): 97.0765, Noisy(x): 105.9592, Filtered(x): 97.1974
...
12948. Actual(x): 388.5226, Noisy(x): 381.7876, Filtered(x): 387.9539
12949. Actual(x): 388.5523, Noisy(x): 398.7570, Filtered(x): 388.0203
12950. Actual(x): 388.5819, Noisy(x): 379.2106, Filtered(x): 388.0179
12951. Actual(x): 388.6115, Noisy(x): 390.3285, Filtered(x): 388.0544
12952. Actual(x): 388.6412, Noisy(x): 388.0000, Filtered(x): 388.0827
...
```