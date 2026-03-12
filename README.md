# Compression Challenge

## Summary of Original Code
The original file implemented a lossless video compressor. It first used FFmpeg to decode the input video into raw frames in gray8 format, meaning each pixel of each frame becomes a single 8-bit grayscale value from 0 to 255. Then, for each pixel (r, c) in each encoded frame (there are 10 total frames by default), it predicted that pixel using only the pixel at the same location in the previous frame (`prior_frame[r,c]`). It then calculated the difference between the true pixel value and that prediction, wrapped the difference into the range from 0 to 255, and arithmetic-coded that difference. This process is called inter-frame prediction because the current frame is being predicted only from an earlier frame in time. 

## Modifications
The primary change made was replacing the inter-frame predictor with a process that combined both inter-frame and intra-frame prediction. My motivation here was thinking of the fact that, in general, knowing more information allows us to make more educated predictions, which (in many cases) leads to a more accurate prediction. From there, I noticed that for any given pixel *x* we’re trying to predict, we have information about at least one of the following: the pixel in the same location within the previous frame, the pixel to the left of *x* in the current frame, and the pixel above *x* in the current frame (because pixels are predicted here from left to right, top to bottom). Furthermore, not only can pixels be very similar between consecutive frames, but also nearby pixels in the same image are often similar. Therefore, the main goal was to take advantage of this additional instance of redundancy to increase the accuracy of the pixel predictions. 

As such, the modified code makes a prediction based on 3 sources: the pixel at the same location in the previous frame, the pixel immediately to the left in the current frame, and the pixel immediately above in the current frame. These are combined by averaging them. In other words, instead of predicting `current(r,c)` only from `prior(r,c)`, the code now predicts `current(r,c)` with something like `(prior + left + above) / 3`. As a result, the predictor now uses both inter-frame redundancy and intra-frame redundancy, which gives us more information to hopefully achieve a more accurate prediction.

### Implementation
To support this, I added a helper function called `combined_predictor`. It calculates the predicted value using `prior`, `left`, and `above`, and for border pixels where `left` or `above` does not exist, it falls back to the previous-frame value. In the encoder, the old residual calculation that was computed from `prior_frame[pixel_index]` was replaced with a residual calculation based on this new predicted value. The arithmetic coder itself stayed the same.

Then, since the predictor now depends on left and above pixels from the current frame, the decoder can no longer reconstruct pixels using only the prior frame. To fix this, I implemented a `reconstructed_frame` buffer for each decoded frame. As each pixel is decoded, it is stored in this buffer, and later pixels use those already reconstructed neighbors for prediction. This ensures losslessness is maintained.

## Results of Modification
Before implementing any modifications, the compression ratio achieved on the `bourne.mp4` file was 2.36, with an average size of 7,014,807 bits/frame. After implementing my modifications, the compression ratio was 3.33, with an average size of 4,988,541 bits/frame. This is a clear improvement since the compression size went down by about 28.9%, while the compression ratio increased by about 41%.  

These results indicate that using both previous-frame information and current-frame information improves the prediction for each pixel, so the residuals are smaller and therefore the arithmetic coder can encode them with fewer bits. 


