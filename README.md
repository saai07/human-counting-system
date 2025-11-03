# How to run

Quickstart (Google Colab)

1. Open the notebook in Google Colab

- Open https://colab.research.google.com and upload `FootFall_Counting.ipynb` using the Upload tab, or open it directly from this repository/GitHub.

2. Upload video file and set the path

- For small test files you can upload directly to the Colab session and then set `video_path` to the uploaded file (example: `/content/FootFall.mp4`).

- For larger files, mount your Google Drive and point `video_path` to the Drive location:

```python
from google.colab import drive
drive.mount('/content/drive')
video_path = '/content/drive/MyDrive/path/to/your_video.mp4'
```

Notes

- Choose a GPU runtime in Colab (Runtime → Change runtime type → GPU) if you want faster processing.
- Before running the notebook cells, ensure `video_path` points to the correct file and that any required model weights (for example `yolov8x.pt`) are available in the session or Drive.


# FootFall Counting with YOLOv8 + BoT-SORT

This project implements an automated people counting system using a combination of object detection, object tracking, and line-crossing analytics. The primary goal is to accurately determine how many people enter or exit a specific area within a video frame — a common requirement in surveillance, retail analytics, and crowd management systems.

## Overview

The system is built around **YOLOv8 (Ultralytics)** for real-time person detection and **BoT-SORT** for robust multi-object tracking. YOLOv8 identifies all visible persons in each frame, while BoT-SORT maintains a consistent ID for every individual across consecutive frames. This ensures that the same person is recognized as they move, even if partial occlusions or temporary detection losses occur.

Once detections and tracking are established, a virtual tripwire (or counting line) is drawn across the frame. The system continuously monitors each tracked person's position relative to this line and determines whether they have crossed it. Based on the crossing direction, the algorithm increments either the **IN** or **OUT** counter.

## Key Features

The pipeline integrates several layers of filtering and validation to ensure stability and reliability:

- **Adaptive tracking points** are selected (e.g., near the feet for tall boxes or mid-body for angled views) to minimize premature crossings
- **Bounding box filters** eliminate detections that are too small or have unrealistic aspect ratios
- **Cooldown mechanism** ensures that each person is counted only once per crossing, avoiding double counts from jitter or hesitation near the line
- **Motion threshold** prevents false triggers caused by minimal pixel-level movements

The output video visually overlays bounding boxes, tracking IDs, and live IN/OUT counts, providing both numerical and visual insights into crowd flow dynamics.

---

## Explanation of Counting Logic

The core of this project lies in its line-crossing counting logic, which intelligently determines when and in which direction a tracked person crosses the defined tripwire. Below is a detailed step-by-step breakdown of how this logic works:

### 1. Detection and Tracking

Each frame is processed through YOLOv8, which detects all persons (class 0). The detected bounding boxes are then passed to BoT-SORT, which assigns a unique tracking ID to every individual. This ID persists across frames, allowing the algorithm to recognize the same person as they move.

### 2. Tripwire Definition

A tripwire (counting line) is defined by two coordinate points:

```python
COUNT_LINE = (x1, y1, x2, y2)
```

This line represents a virtual boundary — for example, a door entrance or passage line. The algorithm treats one side of the line as **"Side A"** and the other as **"Side B"**.

- Crossing from **Side B → Side A** is counted as **IN**
- The reverse is counted as **OUT**


```

### 3. Tracking Point Selection

Rather than using the entire bounding box, the system selects a single representative point for each person — the **tracking point**. This point can be:

- At the **feet** (bottom center of the box) for tall or upright figures, or
- Near the **mid-body** for smaller or slanted boxes

This adaptive logic improves accuracy because the tracking point more closely represents the person's contact with the floor (which is the true crossing reference in most surveillance views).

### 4. Detecting Line Crossings

For every tracked person, the algorithm maintains their **previous position** (`p_prev`) and **current position** (`p_now`). Each frame, it checks:

- Whether the motion distance between these two points is large enough to be meaningful (`MIN_MOTION` filter)
- Whether the segment formed by (`p_prev` → `p_now`) intersects the tripwire line

If both conditions are satisfied, a potential crossing event is detected.

### 5. Determining the Direction of Crossing

To ensure accurate IN/OUT differentiation, the algorithm computes which side of the line the person is on before and after the movement. This is done using a simple orientation test:

- If the person was on **Side B** in the previous frame and on **Side A** in the current frame → counted as **IN**
- If the person moved from **Side A** to **Side B** → counted as **OUT**

This side comparison ensures that only true crossings (not movements along the line) are counted.

### 6. Motion and Cooldown Validation

To enhance robustness:

- **Minimum motion check**: Very small movements near the line (e.g., shifting feet or swaying) are ignored
- **Cooldown period**: After a person triggers a count, a brief cooldown period (measured in frames) is applied to their ID. During this time, further crossings by the same person are ignored to prevent double counting from oscillations

### 7. Updating Counts and Visualization

When a valid crossing is confirmed:

- The corresponding counter (**IN** or **OUT**) is incremented
- A color-coded bounding box (green for IN, orange for OUT) and an ID label are drawn
- The real-time counts are displayed on the video feed as:
  ```
  IN : X
  OUT: Y
  NET: IN - OUT
  ```

### 8. Dependencies

- ultralytics – YOLOv8 object detection
- supervision – Drawing and analytics utilities
- opencv-python – Video and frame processing
- numpy – Numerical computations
- tqdm – Progress bar for frame iteration
- filterpy, lap – Required for BoT-SORT tracking

### 8. Result

The output provides:

- A fully annotated video with bounding boxes, IDs, and tripwire overlay
- Final counts summarizing total entries, exits, and net occupancy changes
- Optional per-frame screenshots and summary statistics (e.g., number of detections filtered, IDs tracked, skipped motion events, etc.)


## Summary

This logic integrates detection, tracking, and geometric reasoning to ensure that only valid, directionally consistent crossings are counted. It is highly adaptable to different camera angles, crowd densities, and motion speeds.
