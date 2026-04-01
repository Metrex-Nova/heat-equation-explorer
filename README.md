# heat-equation-explorer

A single-file browser tool for visualising the separation of variables solution to the 1D heat equation.

```
u_t = alpha^2 * u_xx,    0 < x < 1,  t > 0
u(0,t) = u(1,t) = 0
u(x,0) = phi(x)
```

No server. No build step. Open `heat_equation.html` in a browser.

---

## Usage

1. Enter `phi(x)` in JS syntax (e.g. `x*(1-x)`, `Math.sin(Math.PI*x)`, `x < 0.5 ? x : 1-x`)
2. Set number of Fourier terms, alpha, and t
3. Click **compute**
4. Switch between the two tabs

---

## What the two tabs show

**Error analysis** — plots L2 error vs N (number of terms), where error is measured against a reference solution with 50+ extra terms. A table shows the exact error at each N and percentage reduction from N=1.

**Convergence graphs** — 3D surface plots of u(x,t) built up term by term. If N <= 20, shows all graphs from 1 to N. If N > 20, shows the last 20 (e.g. N=30 shows graphs 11 through 30), since the early terms have already stabilised.

---

## How the math is implemented in JS

**Fourier coefficients**

```
A_n = (2/L) * integral_0^L  phi(x) * sin(n*pi*x/L) dx
```

`scipy.integrate.quad` is not available in the browser, so this integral is computed using Simpson's rule — a numerical integration method implemented manually. 400 subintervals are used per coefficient, which is accurate enough for all smooth phi(x).

```js
function simpson(f, a, b, n=400) { ... }
function computeAn(phi, n, L=1) {
  return (2/L) * simpson(x => phi(x) * Math.sin(n * Math.PI * x / L), 0, L);
}
```

**phi(x) input**

The user types a JS expression string. It is converted into a callable function using `new Function('x', ...)`. This is what allows arbitrary expressions like `x*(1-x)` or `x < 0.5 ? x : 1-x` to be evaluated at any point x.

```js
function makePhi(expr) { return new Function('x', `return ${expr};`); }
```

**Partial sum evaluation**

```
u(x,t,N) = sum_{n=1}^{N}  A_n * exp(-(n*pi*alpha)^2 * t) * sin(n*pi*x)
```

Evaluated pointwise over a grid of x values (for error analysis) or over a meshgrid of x and t values (for 3D surfaces).

**L2 error**

```
error = sqrt( integral_0^1 (u_N - u_ref)^2 dx )
```

Approximated using the trapezoidal rule over 300 equally spaced x points. `u_ref` is the partial sum with `max(N+50, 200)` terms, used as a proxy for the true solution.

**3D surfaces**

Plotly.js takes a 2D array Z where `Z[i][j] = u(x_j, t_i, N)`. The x and t grids are each 35 points, giving a 35x35 mesh per surface. Plotly handles the WebGL rendering.

**Progress bar**

JS is single-threaded, so heavy computation blocks the UI. `await sleep(0)` yields control back to the browser between iterations, allowing the progress bar to repaint. This is the browser equivalent of tqdm in Python.

---

## Dependencies

- Plotly.js 2.27.0 (loaded from cdnjs.cloudflare.com)

Everything else is vanilla JS.

---

## Piecewise phi(x) syntax

Use JS ternary operators:

```
x < 0.5 ? 0 : 1                         step function
x < 0.5 ? x : 1 - x                     triangle wave
x < 0.25 ? 0 : x < 0.75 ? 1 : 0        rectangular pulse
```
