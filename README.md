# FitFlick AI — Photo Validation Gatekeeper

This project is a lightweight image validation script that checks user photos *before* sending them to a virtual try-on model. 

Virtual try-on models require heavy GPU compute and easily break if a user uploads a cropped selfie, a sitting photo, a group shot, or a blurry picture. My goal here was to build a fast gatekeeper (`check_photo`) that filters out bad uploads right away and gives a clear reason why, while letting good full-body standing photos pass through without annoying the user.

---

## How It Works

Instead of running the pose model on every single file right away, `checker.py` checks the basics first so bad files fail fast:

### Step 1: Basic Image Quality (OpenCV)
* **Size check:** Makes sure the photo is at least `500px` tall and `220px` wide. (During testing, I realized a `350px` width rule was too strict because vertical phone photos and WhatsApp forwards are naturally tall and narrow, so I tuned the width down to `220px`.)
* **Brightness check:** Converts the image to grayscale and checks the average pixel brightness (`< 20` fails). This catches very dark or pitch-black photos without rejecting people wearing black clothes.
* **Blur check:** Uses Laplacian variance (`< 50` fails) to catch out-of-focus or shaky camera shots.

### Step 2: Pose & Body Checks (MediaPipe PoseLandmarker)
If the image quality is fine, the photo goes to MediaPipe (`pose_landmarker_heavy.task`):
* **One person only (`num_poses=2`):** Rejects the photo if `0` people are found (like a landscape) or if more than `1` person is in the frame.
* **Full body visible:** Checks that both the left and right ankles have a visibility score of at least `0.5`, and makes sure the feet aren't cut off right at the very bottom edge of the photo (`toe.y > 0.98`).
* **Standing straight:** Calculates the angle between the hip, knee, and ankle. If the knee angle is under `150°`, the person is sitting, crouching, or bending over.
* **Facing the camera:** Checks the horizontal distance between the left and right shoulders (`abs(left_shoulder.x - right_shoulder.x)`). If it's under `0.08`, the person is turned sideways.

---

## Dataset & Tuning (130 Photos)

I collected 130 photos and split them into two separate folders so I wouldn't overfit my cutoff numbers to the final test set:
* **`dataset/tune` (30 photos — 10 good, 20 bad):** Used this smaller set to tweak my brightness, blur, and width numbers.
* **`dataset/test` (100 photos — 30 good, 70 bad):** Ran this once at the end to get my final scores. The 70 bad photos are split across 5 common mistakes: cropped selfies, sitting/wrong angles, couples/multiple people, dark/blurry quality, and empty backgrounds with no people.

---

## Final Test Results (100 Photos)

| Metric | Result | Percentage | What This Means |
| :--- | :---: | :---: | :--- |
| **Good photos wrongly rejected** *(False Rejection Rate)* | 2 / 30 | **6.7%** | 28 out of 30 good photos passed on the first try. |
| **Bad photos that slipped through** *(False Acceptance Rate)* | 2 / 70 | **2.9%** | Caught 68 out of 70 bad photos before they could waste GPU time. |

### Where It Made Mistakes:
* **The 2 rejected good photos (`good (12).png`, `good (14).png`):** In both of these, the person's full body was visible, but their shoes were touching the very bottom edge of the frame, which triggered my bottom-border cutoff rule.
* **The 2 bad photos that passed (`bad (20).jpeg`, `bad (73).png`):** Borderline cases where MediaPipe was still able to guess the full body landmarks cleanly enough to pass the cutoffs.

---

## Can This Run on a Phone Before Upload? (Google ML Kit)

Yes. Google's **ML Kit Pose Detection** runs on both Android and iOS and uses the exact same 33-body-point layout as MediaPipe. Because it runs right on the phone's processor at around 30 FPS, we can run these exact same angle and visibility checks on the user's phone *before* the photo ever uploads to the server. 

This saves mobile data, cuts server costs, and lets the app coach the user live on the camera screen (like telling them *"Step back so your ankles show"* before they even snap the picture). The main trade-off is that the mobile model is lighter than the `heavy` desktop model used here, so landmark points might jitter slightly more in bad lighting on cheaper phones.

---

## How to Run the Code

1. **Install the required libraries:**
   ```bash
   pip install -r requirements.txt
