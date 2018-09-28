  <h1 align="center">Obejct Detection - Cuda</h1> 
  <p align="center">
  <img src="https://img.shields.io/badge/License-MIT-blue.svg">
  </p>


> Implementation of a motion detection solution with OpenCV and Cuda, as part of the Intelligent Industrial System Course.

### Part 1: Convert an RGB color image to HSV, without using an external library.

#### Here's the algorithm :
The R,G,B values are divided by 255 to change the range from 0..255 to 0..1:<br>
R' = R/255<br>
G' = G/255<br>
B' = B/255<br>
Cmax = max(R', G', B')<br>
Cmin = min(R', G', B')<br>
Δ = Cmax - Cmin<br>
Hue calculation:<br>
http://www.rapidtables.com/convert/color/rgb-to-hsv/hue-calc.gif<br>
Saturation calculation:<br>
http://www.rapidtables.com/convert/color/rgb-to-hsv/sat-calc.gif<br>
Value calculation: V = Cmax<br>

The algorithm running on both the CPU and the GPU has been implemented in both cases, to facilitate understanding of the steps to follow when paralleling in Cuda.

```c++
void rgbToHSV(Mat frame) {
	Vec3b hsv;
	for (int rows = 0; rows < frame.rows; rows++) {
		for (int cols = 0; cols < frame.cols; cols++) {
			float blue =  frame.at<Vec3b>(rows, cols)[0] / 255.0; // blue
			float green = frame.at<Vec3b>(rows, cols)[1] / 255.0; // green
			float red = frame.at<Vec3b>(rows, cols)[2] / 255.0; // red

			float maximum = MAX3(red, green, blue);
			float minimum = MIN3(red, green, blue);

			float delta = maximum - minimum;
			uchar h = HueConversion(blue, green, red, delta, maximum);
			hsv[0] = h / 2;
			uchar s = (delta / maximum) * 255;
			hsv[1] = s;
			float v = (maximum) * 255;
			hsv[2] = v;

			frame.at<Vec3b>(rows, cols) = hsv;
		}
	}
}
```

Also, here is an example of a way to calculate the hue value in c++ :

```c++

uchar HueConversion(float blue, float green, float red, float delta, float maximum) {
	uchar h;
	if (red == maximum) { h = 60* (green - blue) / delta; }
	if (green == maximum) { h = 60 * (blue - red) / delta + 120; }
	if (blue == maximum) { h = 60 * (red - green) / delta + 240; }
	if (h < 0) { h += 360; }
	return h;
}
```
> The Hue value you get needs to be multiplied by 60 to convert it to degrees on the color circle. If Hue becomes negative you need to add 360 to, because a circle has 360 degrees.

###### `MAX`/`MIN` and `MAX3`/`MIN` functions were made to calculate highest and lowest value between 2 or 3 parameters like this :
```c++
#define MIN(a,b)      ((a) < (b) ? (a) : (b))
#define MAX(a,b) ((a) > (b) ? (a) : (b))
#define MIN3(a,b,c)   MIN((a), MIN((b), (c)))
#define MAX3(a,b,c) MAX((a), MAX((b), (c)))
```

### Part 1: Convert an HSV frame to Sobel Filter, without using an external library.

This part has been made in Cuda, to get a quicker result.

```c++
__global__ void parallelSobelFilter_kernel(uchar* inputImage, uchar* outputImage, int width, int height)
{
	int gIndex = blockIdx.x  * blockDim.x + threadIdx.x; // global index
	int tIndex = gIndex - width; // top index
	int bIndex = gIndex + width; // bottom index

	if (tIndex < 0 || bIndex>(width*height)) {
		return;
	}

	int gradientX =
		inputImage[tIndex - 1] * Gx[0][0] + // left top
		inputImage[gIndex - 1] * Gx[1][0] + // left middle
		inputImage[bIndex - 1] * Gx[2][0] + // left bottom
		inputImage[tIndex + 1] * Gx[0][2] + // right top
		inputImage[gIndex + 1] * Gx[1][2] + // right middle
		inputImage[bIndex + 1] * Gx[2][2];  // right bottom

	int gradientY = 
		inputImage[tIndex - 1] * Gy[0][0] + // left top
		inputImage[tIndex] * Gy[0][1] + // middle top
		inputImage[bIndex - 1] * Gy[2][0] + // left bottom
		inputImage[tIndex + 1] * Gy[0][2] + // right top
		inputImage[bIndex] * Gy[2][1] + // middle bottom
		inputImage[bIndex + 1] * Gy[2][2];  // right bottom

	gradientX = gradientX * gradientX;
	gradientY = gradientY * gradientY;
	float approxGradient = sqrtf(gradientX + gradientY + 0.0);
	if (approxGradient > 255) { 
		approxGradient = 255;
	}
	outputImage[gIndex] = (uchar)approxGradient;
}
```

This part of code has been generated by a kernel, which was called in our main CPU file :

```c++
extern "C" cudaError_t parallelSobelFilter(Mat *inputImage, Mat *outputImage) {
	cudaError_t status;
	uchar *inputSobel, *outputSobel;
	// 0. Define block dimension
	int BLOCK_COUNT = iDivUp((inputImage->cols * inputImage->rows), BLOCK_SIZE);
	// 1. Define input/output image sizes
	uint imgSize = inputImage->rows * inputImage->step1();
	uint gradientSize = inputImage->rows * inputImage->cols * sizeof(uchar);
	// 2. Allocate memory space for both matrix on gpu
	status = cudaMalloc(&inputSobel, imgSize);
	if (status != cudaSuccess)
	{
		fprintf(stderr, "cudaMalloc failed");
		goto Error;
	}
	cudaMalloc(&outputSobel, gradientSize);
	if (status != cudaSuccess)
	{
		fprintf(stderr, "cudaMalloc failed");
		goto Error;
	}
	// 3. Send matrix(A) to gpu
	status = cudaMemcpy(inputSobel, inputImage->data, imgSize, cudaMemcpyHostToDevice);
	if (status != cudaSuccess)
	{
		fprintf(stderr, "cudaMemcpy failed");
		goto Error;
	}
	// 4. Treat matrix in sobel filter kernel
	parallelSobelFilter_kernel<<<BLOCK_COUNT, BLOCK_SIZE>>>(inputSobel, outputSobel, inputImage->cols, inputImage->rows);
	// 5. Wait for the kernel to end
	status = cudaDeviceSynchronize();
	if (status != cudaSuccess)
	{
		fprintf(stderr, "cudaDeviceSynchronize failed");
		goto Error;
	}
	// 6. Transfer result matrix to output image
	status = cudaMemcpy(outputImage->data, outputSobel, gradientSize, cudaMemcpyDeviceToHost);
	if (status != cudaSuccess)
	{
		fprintf(stderr, "cudaMemcpy failed");
		goto Error;
	}
	// 7. Free matrix in memory
	cudaFree(inputSobel);
	cudaFree(outputSobel);
Error:
	cudaFree(inputSobel);
	cudaFree(outputSobel);

	return status;
}

```


#### This project is currently under development. If you have good ideas, do not hesitate to contribute! 😊
