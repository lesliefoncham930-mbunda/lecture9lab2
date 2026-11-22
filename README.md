# lecture9lab2
Lab 2: 2D Convolution & FIR Filters (Blur, Sharpen, Edges)
1) Delta image & impulse response (Figure 1)

A delta image is all zeros except for a single 1 in the centre (51,51). It is the 2D version of a unit impulse δ[m,n]. When you convolve a filter with an impulse, the output is the filter kernel itself:

δ[m,n] * h[m,n] = h[m,n]

The figure shows this. The output is a small 3×3 yellow square in the centre with value 1/9 ≈ 0.111 (see the colorbar), and everything else is 0. The kernel is the impulse response of the filter. Because the filter has a finite number of taps (9), it is a 2D FIR filter.

2) Low-pass: box vs Gaussian, separability (Figure 2)
Box 3×3: each pixel becomes the average of its 3×3 neighbourhood. The blur is slight, and fine texture and noise are reduced.
Box 7×7: averages 49 pixels, so the blur is much stronger. Edges spread out and detail on the peppers and garlic is lost. Box filters can also produce slightly blocky or streaky artifacts.
Gaussian (7×7, σ = 1.2): a weighted average. The centre pixel gets the most weight and the weight falls off smoothly with distance. The result is a moderate blur that looks more natural and keeps edges better than the 7×7 box.

The Gaussian kernel is built as h_gauss = g1d' * g1d, the outer product of two 1D Gaussians, so the filter is separable.

All of these are low-pass filters. Their kernel values are all positive and sum to 1, so they keep the average brightness (the DC component) and attenuate high spatial frequencies such as edges, texture and noise.

3) Unsharp masking (Figure 3)
Blur: a Gaussian low-pass version of the image.
Mask = I − I_blur: removing the low frequencies leaves the high-frequency detail (edges and texture). The mask is centred around 0. The montage shows negative values as black, so the mask looks almost completely black, with faint bright lines along the edges.
Sharpened = I + gain·mask (gain = 1, clipped to [0,1]). Adding the detail back boosts the edges, so the pepper outlines, the garlic and the cloth folds look crisper than in the original.

The whole process is a high-boost filter: I_sharp = (1 + k)·I − k·I_blur. Raising the gain sharpens more but also amplifies noise and creates halos around edges.

4) Edges: Sobel & Laplacian (Figure 4)
Sobel Gx / Gy: first-derivative (gradient) filters with smoothing built in. On a gray background (0 after mat2gray), edges show up as light or dark lines depending on the direction of the intensity change. Each image highlights edges in one direction.

Note: fspecial('sobel') returns [1 2 1; 0 0 0; -1 -2 -1], which responds to horizontal edges (a vertical gradient). So the image labelled “Gx” actually shows horizontal edges and “Gy” shows vertical ones. It is worth saying this in your report.

Gradient magnitude √(Gx² + Gy²): combines both directions and does not depend on edge orientation. The background is black and the outlines of every pepper and garlic bulb are bright.
Laplacian: a second-derivative filter that is isotropic (responds to edges in every direction). It produces thin edges with zero-crossings, but it is very sensitive to noise, so the image looks grainy.

Edge filters are high-pass filters. Their coefficients sum to 0, so flat regions give 0 output and only intensity changes give a response.

5) Correlation vs convolution (console output)

Max difference (conv2 vs imfilter with conv): 3.33067e-16

Convolution flips the kernel by 180° before sliding it over the image. Correlation does not flip it. imfilter uses correlation by default, and 'conv' makes it flip the kernel.
The difference is about 3.3×10⁻¹⁶, which is floating-point rounding (close to machine epsilon). The two results are effectively identical.
The 3×3 box kernel is symmetric, so flipping it changes nothing and correlation and convolution give the same result anyway. For an asymmetric kernel such as Sobel, the two would differ (the sign of the output flips).
6) Boundary handling (Figure 5)

Near the image border, part of the 7×7 kernel lands outside the image, so the missing pixels have to be filled in somehow:

replicate: repeats the outermost row and column of pixels.
symmetric: mirrors the image across the border.
circular: wraps around, treating the image as periodic, so the top edge borrows pixels from the bottom and the left edge from the right.

In the interior all three results are identical. They differ only in a band about 3 pixels wide around the border (the kernel radius), so the differences are hard to see in the thumbnail. Replicate and symmetric look almost the same. Circular can produce visible artifacts where two different edges get mixed, for example the dark cloth at the bottom bleeding into the top rows. Zero padding, the default in conv2 and imfilter, would instead give a dark border.

Reflections

1) Why is a Gaussian preferred over a large box low-pass filter?
In the frequency domain, a box filter’s response is a sinc function. It has a fairly sharp main lobe and many sidelobes that alternate in sign. As a result, a box filter lets some high frequencies through, can invert some frequencies, and causes ringing and blocky artifacts. A box is also not rotationally symmetric, so it blurs diagonals differently from horizontal and vertical directions. The Fourier transform of a Gaussian is another Gaussian: smooth, monotonically decreasing, with no sidelobes and no ringing. The Gaussian is also isotropic (blurs equally in every direction), weights nearby pixels more than distant ones (so it keeps edges better), and has the best possible trade-off between spatial and frequency localization. σ controls the cutoff in a predictable way. This is why the Gaussian blur in Figure 2 looks smoother and more natural than the 7×7 box.

2) What does separability do for computational cost?
A filter is separable when its 2D kernel can be written as the outer product of two 1D kernels: h[m,n] = h₁[m]·h₂[n]. You can then filter the rows with the 1D kernel and filter the columns of that result with the other 1D kernel, and get exactly the same output as the 2D filter.

Full 2D K×K kernel: K² multiplications per pixel
Two 1D passes: 2K multiplications per pixel

For the 7×7 Gaussian that is 49 vs 14 operations per pixel, about 3.5× faster. The saving grows with kernel size: for 31×31 it is 961 vs 62, about 15×. Gaussian and box filters are both separable. (MATLAB’s imfilter checks for separability automatically and uses two passes when it can.)

3) How do boundary modes change corners and edges?
Boundary handling only affects pixels within the kernel radius of the border, which is 3 pixels for a 7×7 kernel. The corners are affected most, because the kernel is outside the image in two directions there.

Zero padding: pulls border pixels toward black, giving a dark frame around the image.
Replicate: extends the edge values outward. There is no dark border, but the extended edge values are given extra weight, which can cause slight streaking.
Symmetric: mirrors the image content. This usually gives the most natural, seamless result, because the extended region is continuous with the image.
Circular: wraps the image around, so opposite edges get mixed (for example, bottom content appears at the top). This causes artifacts whenever opposite edges differ. It matches FFT-based filtering, which assumes the image is periodic.
