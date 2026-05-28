# ProtoCode
A purely morphological computer vision system that reads hand-drawn logic gate circuits from a physical education kit and outputs a boolean expression. No neural networks: designed to be lightweight enough to run on low-end devices and fail in predictable, debuggable ways.

> Note: The documentation in this README was written with the help of Claude (Anthropic). The project, the concept, the physical kit design, the algorithm, and all the engineering decisions documented here: is my own work.

# What is this?
ProtoCode is a STEAM education aid I built during my internship. The physical kit has a whiteboard with a grid, a set of tiles with logic gate symbols on them, and a marker. A kid draws the connecting wires on the whiteboard, places the gate tiles over the drawn symbols, takes a photo, and the algorithm reads the circuit and outputs the boolean expression.

# Why does this exist?
I was given an open brief at my internship to make a STEAM product, no further direction. So I had to invent the problem and the solution. I landed on logic gates because they're one of those topics that's genuinely hard to make hands-on. Most tools are either pure software simulations or expensive hardware kits. This sits in between, physical enough to feel real, simple enough that the kit is just a whiteboard and some tiles.

# How it works
The algorithm runs in stages. Each stage exists because the previous one leaves something behind that the next stage can't handle cleanly.
## 1. Orientation correction
A photo can be taken at any rotation. Before anything else, the algorithm detects one of the four corner ArUco markers on the whiteboard and reads which way it's pointing. Based on that, it rotates the entire image to upright — 0°, 90°, 180°, or 270°. Everything after this assumes the board is the right way up.
## 2. Perspective warp
A photo taken from any angle will have perspective distortion — the board looks like a trapezoid instead of a rectangle. The algorithm finds all four corner markers, maps their positions to a perfect rectangle, and flattens the image to a clean top-down view. From this point on, the board fills the entire image at a known size.
<img width="1790" height="565" alt="image" src="https://github.com/user-attachments/assets/f99b0731-cbcd-463c-af1c-666ed5a04583" />

## 3. Gate detection and erasure
Each logic gate tile has an ArUco marker on it with a unique ID that maps to a gate type (AND=3, OR=5, XOR=7, etc). The algorithm detects all gate markers in one pass, records each gate's type and center position, then whites out each gate's bounding box. This leaves only the hand-drawn wires on an otherwise clean white board — which is exactly what the rest of the pipeline needs.
<img width="1389" height="517" alt="image" src="https://github.com/user-attachments/assets/f773cffa-b672-4e8f-9f04-6e894352c20c" />

## 4. Adaptive thresholding
The image gets converted to grayscale and thresholded to black and white — wires become black, background becomes white. A global threshold fails here because lighting is never perfectly even across the board. Instead, the algorithm uses adaptive thresholding with a circular kernel, which computes a local brightness reference for each pixel based on its immediate neighborhood. This handles the patches and shadows that a fixed threshold would misread as lines.
## 5. Morphological opening and closing
After thresholding, the image has the drawn circuit lines but also the whiteboard grid and small artifacts from uneven lighting. Opening (erosion followed by dilation) first connects small gaps in the drawn circuit lines, then closing (dilation followed by erosion) removes the thinner whiteboard grid lines and leftover artifacts. Opening happens before closing deliberately: doing it the other way around would disconnect the circuit lines completely.
<img width="1790" height="450" alt="image" src="https://github.com/user-attachments/assets/257c2af4-7395-41a7-be45-09246129bead" />

> ⚠️ Polarity note: OpenCV's morphological operation names assume white foreground on black background. In this pipeline the lines are black on a white background. So the operations behave in the opposite way to what their names suggest. Opening here connects the lines rather than removing small objects, and closing removes thin noise rather than filling gaps. Keep this in mind if you're reading the code.
## 6. Median smoothing
The drawn lines at this point have jagged, irregular edges — a natural result of hand-drawing. If you skeletonize a jagged stroke directly, you get "webbing": a mess of branching 1-pixel lines instead of a clean single path. Median blur smooths the edges of each stroke into clean rounded shapes before skeletonization, which prevents webbing from forming in the first place.
## 7. Skeletonization
Each wire stroke gets reduced to a single-pixel-wide path using medial axis skeletonization. This is the step that turns thick drawn lines into topological wire paths the graph traversal can actually use.
<img width="1790" height="450" alt="image" src="https://github.com/user-attachments/assets/319d2108-ec47-40b8-878c-b3d27e334dc3" />

## 9. Feature extraction and graph traversal
The skeleton gets analyzed pixel by pixel. Each pixel's neighbor count determines its role: 1 neighbor = endpoint (where a wire starts or terminates), 3+ neighbors = junction (where wires meet or split). These features get cleaned up to remove false positives left over from imperfect skeletonization, then the algorithm traverses the resulting graph - following connections from gate outputs to gate inputs — and recursively builds the boolean expression.
<img width="1365" height="989" alt="image" src="https://github.com/user-attachments/assets/4e2e9231-f854-4ac6-b43e-ae62513e30d9" />
<img width="1365" height="989" alt="image" src="https://github.com/user-attachments/assets/9fcfe739-b140-499a-bdb8-493bf573386c" />
<img width="1365" height="989" alt="image" src="https://github.com/user-attachments/assets/cf823fa8-cf78-4e20-b38d-c61e433d7637" />

# The Fights
This section documents what actually went wrong during development and how I responded to it. The pipeline in the section above looks clean in hindsight — this is what it looked like getting there.
## Fight 1 - Thresholding
The first major problem was patches in the thresholded image. Certain areas of the board would threshold incorrectly, reading background as wire or wire as background, because the lighting across the board was never perfectly even.
The first attempt was simple (global) thresholding. It failed immediately under any real lighting condition: a fixed brightness cutoff can't handle a board that's brighter in one corner than another.

Switched to adaptive thresholding, which computes a local brightness reference per pixel. Better, but still not right, the standard square kernel was sampling pixels in all directions equally, which meant it was being influenced by the horizontal and vertical grid lines of the whiteboard when computing the local reference for nearby wire pixels.

Tried a cross-shaped kernel next, hoping it would isolate horizontal and vertical lines better. Then a circular kernel as an experiment. The circular kernel ended up working best - it samples a natural neighborhood around each pixel without any directional bias, giving the cleanest local brightness estimate for hand-drawn content on a grid background.
## Fight 2 - Morphology and the grid
After thresholding, the circuit lines were there, but so was the whiteboard grid and various artifacts from lighting. The grid lines were thinner than the drawn circuit lines, which gave something to work with morphologically.

The sequencing of opening and closing took some working out. The intuitive order (close first to fill gaps, open to remove noise) actually disconnected the circuit lines. The correct order turned out to be open first (to connect small gaps in drawn lines) then close (to remove the thinner grid lines and artifacts). Getting that order wrong broke the circuit topology entirely.
## Fight 3 - Getting to a clean skeleton
This was the longest fight. The goal was a single-pixel-wide wire path, a skeleton the graph traversal could actually walk. The problem was webbing: instead of clean single paths, skeletonization on raw morphological output produced branching tangles of 1-pixel lines at every imperfection in the drawn stroke.
Tried 
- Gaussian blur iterations
- blur, threshold, repeat

to smooth strokes before skeletonization. The lines would disappear or merge if the kernel was too large, break apart if it was too small. Tried multiple skeletonization approaches: 
- OpenCV's iterative erosion method
- skimage.skeletonize
- skimage.thin
- finally skimage.medial_axis.

Tried a separate heal_skeleton function to bridge gaps after the fact.
Median blur turned out to be the right smoothing step, it preserves edge sharpness better than Gaussian while still rounding off the jagged stroke edges that cause webbing. Combined with the right morphological preparation, this gave a clean enough skeleton for the feature extraction to work.
## The thing I'm quietly proud of
After skeletonization, there were still false junctions and endpoints: artifacts of imperfect skeletonization rather than real circuit topology. I designed a cleanup filter that identifies and removes these based on proximity rules: a junction too close to an endpoint is a bend not a junction, two endpoints too close together are a continuous line not two terminals.

This wasn't from a textbook. It came from staring at the output long enough to understand exactly how the skeleton was lying to me.
