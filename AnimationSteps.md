## **🟢 Animation & Audio Lab: Red Light, Green Light Doll**

This lab focuses on the fundamental workflow for creating and controlling animations in Unity using the **Animator** state machine, **AudioSource** timing, and C\# **Coroutines**.

### **Stack & Prerequisites**

* **Unity:** 2022.3 LTS or newer  
* **Components:** Animator, AudioSource  
* **Scripting:** C\# Coroutines

# **🧱 Block 1 — Creating the Animation Clip (The Movement)**

**Goal:** Create a reusable .anim asset that defines the doll's movement (the head rotation for the "Red Light" phase).

### **Editor Steps**

1. **Create Object:** In the Hierarchy, create a **3D Cube** (Right-click $\\rightarrow$ 3D Object $\\rightarrow$ Cube). This will stand in for your doll's head/body.  
2. **Open Animation Editor:** Select the Cube. Go to **Window** $\\rightarrow$ **Animation** $\\rightarrow$ **Animation** (or press **Ctrl \+ 6**).  
3. **Create Clip:** In the Animation window, ensure the **Cube is selected**, then press the **Create** button. Name the file HeadRotation and save it.  
   * *(Unity automatically adds an **Animator** component to the Cube and creates an **Animator Controller** asset.)*  
4. **Record Movement:** Press the **Record** button (the red circle $\\text{\\textcircled{}}$).  
   * **Keyframe 1 (0:00):** Set the Cube's **Y Rotation** to 0 (Looking Forward).  
   * **Keyframe 2 (1:00):** Move the timeline slider to **1:00** (one second). Set the **Y Rotation** to 180 (Looking Back).  
   * **Keyframe 3 (2:00):** Move the timeline slider to **2:00**. Set the **Y Rotation** back to 0 (Looking Forward).  
5. **Stop Recording:** Press the **Record** button again.

**Why:** This creates a clean, repeatable animation asset that the Animator Controller can trigger on command.

**Mini-Test:** Press **Play** in the **Animation** window. The Cube should smoothly rotate 180 degrees and return over two seconds.

**Common Mistakes:**

* Forgetting to set the Cube's layer to **"Nothing"** before recording can accidentally keyframe unrelated layer weights.  
* Leaving the timeline slider on an arbitrary time when starting a new movement.

# **🧱 Block 2 — Animator Controller (The Logic Map)**

**Goal:** Configure the state machine to use a **Boolean parameter** to toggle between the **Idle** (Green Light) state and the **Rotation** (Red Light) state.

### **Editor Steps**

1. **Open Animator:** Select the Cube. Go to **Window** $\\rightarrow$ **Animation** $\\rightarrow$ **Animator** (or **Ctrl \+ 7**). You should see the HeadRotation state.  
2. **Create Parameter:** In the **Parameters** tab (top left of the Animator window), click the **\+ button** and create a **Bool** parameter named **ROTTE** (case-sensitive).  
3. **Create Idle State:** Right-click in the Animator graph $\\rightarrow$ **Create State** $\\rightarrow$ **Empty**. Name this new state **Idle**.  
4. **Transition (Idle** $\\rightarrow$ **Rotation):**  
   * Right-click on the **Idle** state $\\rightarrow$ **Make Transition** $\\rightarrow$ click on the **HeadRotation** state.  
   * Select the transition arrow. In the Inspector, **uncheck "Has Exit Time"** and set the **Condition** to **ROTTE is true**.  
5. **Transition (Rotation** $\\rightarrow$ **Idle):**  
   * Right-click on the **HeadRotation** state $\\rightarrow$ **Make Transition** $\\rightarrow$ click back on the **Idle** state.  
   * Select the transition arrow. In the Inspector, **uncheck "Has Exit Time"** and set the **Condition** to **ROTTE is false**.

**Why:** The Animator handles the smooth blending between animations. We use a Boolean because the doll is in one of two distinct states: **Green Light** (ROTTE \= false) or **Red Light** (ROTTE \= true).

**Mini-Test:** Press **Play**. Open the **Animator** window. Manually check and uncheck the **ROTTE** parameter. The Cube should instantly stop/start rotating as you toggle it.

**Common Mistakes:**

* Forgetting to **uncheck "Has Exit Time"** on the transitions, which causes a delay between the code flipping the boolean and the animation actually starting.

# **🧱 Block 3 — C\# Scripting (The Conductor)**
**Goal:** Create the script that implements the infinite game logic: play audio, wait for it to stop, and then toggle the ROTTE boolean.

### **Editor Steps**

1. Create a new folder named Scripts.  
2. Create a C\# script named DollBehaviour.cs.

------------------------------------------------------------------------------------------------

# Code for Player Movemrnt #

You’re right—let’s make it dead simple.

## Goal

Track **only the Right Hand Anchor**. If the right hand moves **up or down** by a threshold in **Y**, move the player **forward**. No boxes, no triggers.

---

## 1) Minimal script

Create `RightHandYForward.cs` and paste:

```csharp
using UnityEngine;

public class RightHandYForward : MonoBehaviour
{
    [Header("Assign these in Inspector")]
    public Transform rightHandAnchor;   // OVRCameraRig/TrackingSpace/RightHandAnchor  (or Interactors/Hand)
    public Transform player;            // OVRCameraRig/TrackingSpace  (or your Player root)

    [Header("Tuning")]
    public float detectionRange = 0.15f;   // meters above/below center to start moving
    public float moveSpeed = 2f;           // meters/second
    public bool continuous = true;         // true = move every frame while outside threshold

    [Tooltip("Used only if continuous=false (debounce between pulses)")]
    public float minTriggerInterval = 0.12f;

    private float centerY;
    private float lastTriggerTime;
    private CharacterController cc;

    void Awake()
    {
        if (player) cc = player.GetComponent<CharacterController>();
    }

    void Start()
    {
        if (rightHandAnchor) centerY = rightHandAnchor.position.y;  // capture neutral height at start
    }

    void Update()
    {
        if (!rightHandAnchor || !player) return;

        float deltaY = rightHandAnchor.position.y - centerY;
        float abs = Mathf.Abs(deltaY);

        if (continuous)
        {
            if (abs >= detectionRange)
            {
                MoveForward();
                // re-center so small up/down keeps you moving
                centerY = rightHandAnchor.position.y;
            }
        }
        else
        {
            if (abs >= detectionRange && (Time.time - lastTriggerTime) >= minTriggerInterval)
            {
                MoveForward();
                lastTriggerTime = Time.time;
                centerY = rightHandAnchor.position.y;
            }
        }
    }

    public void CalibrateNow()  // call this if you want to recenter during play
    {
        if (rightHandAnchor) centerY = rightHandAnchor.position.y;
    }

    private void MoveForward()
    {
        Vector3 f = player.forward; f.y = 0f;
        if (cc != null) cc.Move(f.normalized * moveSpeed * Time.deltaTime);
        else            player.position += f.normalized * moveSpeed * Time.deltaTime;
    }
}
```

---

## 2) Hook it up (2 assignments only)

1. **Create Empty** → name `RightHandYDriver` → add `RightHandYForward`.
2. In the **Inspector**:

   * **Right Hand Anchor** → `OVRCameraRig/TrackingSpace/RightHandAnchor`

     * (If you’re using the Interaction prefab instead, use `RightInteractors/Interactors/Hand`.)
   * **Player** → `OVRCameraRig/TrackingSpace`  *(this is the safest object to move)*

**Tuning for quick test**

* `detectionRange = 0.15`
* `moveSpeed = 2`
* `continuous = true`

Press **Play** → raise or lower your right hand by ~15–20 cm → the rig moves forward.

---

## 3) If it doesn’t move, check only these 3 things

1. **Right hand reference** is the real tracked object:

   * Prefer `TrackingSpace/RightHandAnchor` (or `Interactors/Hand`), **not** a DataSource.
2. **Player** is `TrackingSpace` (or your CharacterController root).
3. You’re actually moving the hand **≥ detectionRange** in Y.

---

## (Optional) Want the visible boxes later?

Use your cubes again; the transparency issue is just material setup in URP:

* Shader: **Universal Render Pipeline/Lit** (or **Unlit**)
* **Surface Type: Transparent**
* Lower the **Base Color Alpha** (e.g., 0.25)
* Apply the material to each cube’s **Mesh Renderer > Element 0**.

---

When this right-hand version works, say the word and I’ll flip it to **both hands** in one clean script.
