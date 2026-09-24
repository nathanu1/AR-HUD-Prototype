# Building Wearable Glasses With a UX Overlay — A Detailed Build Breakdown

This is a full-stack breakdown of what it takes to build a pair of see-through glasses that paint a user interface over the real world. It covers optics, display, compute, sensors, power, mechanical design, the software/UX layer, and the order you should actually build things in.

> "AR glasses" spans an enormous difficulty range. A monocular heads-up display that floats notifications and navigation arrows in your periphery. A binocular, world-locked, occlusion-capable display with the image welded to physical space is a problem that billion-dollar companies are still fighting. The single biggest determinant of how hard your build is, is **the optical combiner** — getting an image in front of the eye while still seeing the world. 

---

## 0. Build Levels

| Tier | What it does | Optics | Compute |  
|---|---|---|---|---|
| **1 — Glanceable HUD** | Monocular overlay: notifications, time, nav arrows, telemetry, teleprompter. Image floats in space, *not* locked to the world. | Birdbath or simple magnifier + beamsplitter | Phone or small SBC, MCU for sensors | **Yes.** Best starting point. |
| **2 — Head-locked binocular** | Two displays, stereo image, still floats with your head. Bigger virtual screen, watch-video / multi-widget UX. | Two birdbath modules **or** sourced waveguide modules | SBC (Raspberry Pi CM-class / Jetson) | Hard but doable. Alignment is the pain. |
| **3 — World-locked spatial AR** | Objects anchored to physical space, survive head movement, can hide behind real things (occlusion). | Waveguides + precise calibration | Jetson-class + SLAM | **Not realistically DIY** at good quality. Source a dev kit instead. |

---

## 1. Product Definitions


- **Monocular or binocular?** Monocular halves your optics, compute, and power budget and is far easier to align. It's also genuinely useful for a HUD.
- **See-through (optical) or pass-through (camera)?** Optical see-through (light passes through, image overlaid) is what "glasses" usually means and is lighter. Pass-through (cameras feed a closed display) is easier to render to but is really a small VR headset.
- **Field of view (FOV).** A HUD lives happily at 15–25° diagonal. Immersive AR wants 40°+, which is exponentially harder. 
- **Brightness target.** Indoor only? A few hundred to ~1,000 nits to the eye is fine. Usable outdoors in daylight needs far more at the panel — see §3.
- **World-locked or head-locked?** This decides whether you need SLAM (cameras + heavy compute) or just an IMU.
- **Tethered or standalone?** Tethering to a phone or a pocket compute puck moves the battery and the heat off your face. Standalone is the dream and the thermal/weight nightmare.
- **Runtime, weight, and budget.** Set a gram budget and a dollar budget now. Comfortable all-day glasses are roughly under ~70 g; HUD prototypes commonly land at 100–200 g and that's okay for a bench build.

A good first-build target: *monocular, optical see-through, ~18° FOV, indoor brightness, head-locked, tethered to a phone, ~$300–$600 in parts.*

---

## 2. System architecture (the block diagram)

```
                 ┌─────────────────────────────────────────────┐
                 │                  GLASSES                      │
   real world →  │  [Combiner] ← [Lens] ← [Microdisplay]         │
      light      │      ↓ (to eye)            ↑ MIPI/HDMI         │
                 │                      [Display driver board]    │
                 │  [IMU 9-DoF] [Camera] [ToF] [ALS] ─ I²C/CSI ─┐ │
                 │  [Mic] [Touch/buttons]                       │ │
                 │  [MCU: sensor hub + power mgmt]              │ │
                 │  [LiPo + BMS + USB-C PD]                     │ │
                 └──────────────────┬───────────────────────────┘ │
                                    │ USB-C / BLE / Wi-Fi          │
                 ┌──────────────────▼───────────────────────────┐ │
                 │   HOST (phone or SBC: render + app logic) ◄───┘ │
                 │   compositor → overlay UI → display frames       │
                 └──────────────────────────────────────────────┘
```

The recurring design decision is **where the split is**: which work happens on the glasses (always: drive the panel, read sensors, manage power) vs. on a host (often: render the UI, run apps, do any vision/SLAM). For Tier 1–2, "render on the host, stream frames to the panel, read sensors from a microcontroller" is the sane architecture.

---

## 3. The optical system — the hard part

Difficulty within itself; making the optics, The combiner merges your tiny bright image with the world.

### Combiner options, ranked by DIY-friendliness

1. **Simple magnifier + plate/pellicle beamsplitter .** A lens collimates the microdisplay; a partially-reflective flat (a 50/50 beamsplitter plate, or a pellicle) bounces it into the eye while you see through it. Cheapest, most forgiving, smallest FOV, dimmest. Great for learning.
2. **Birdbath optics** Light from the display hits a beamsplitter at ~45°, reflects to a concave ("birdbath") spherical mirror that collimates and magnifies it, then passes back through the beamsplitter to the eye. Off-the-shelf beamsplitter cubes/plates and small concave mirrors make this buildable. It's bulkier and you lose a lot of light passing the beamsplitter twice (often only ~10–25% efficiency), but image quality and FOV are good. This is the standard hobbyist and many early-commercial choice.
3. **Reflective freeform / prism.** A custom-shaped prism folds the path. Larger FOV, but you need the specific part — you're sourcing, not fabricating.
4. **Waveguides.** A thin glass/plastic slab; light is coupled in, bounces by total internal reflection, and is coupled out to the eye via gratings. Sleek and thin, but fabrication needs nanometer-scale gratings and clean-room processes. Throughput is brutal — only on the order of **a few percent** of the light from the engine reaches the eye, which is why waveguide builds demand extremely bright panels. You *can buy* waveguide evaluation modules; you cannot make good ones at home.
5. **Retinal/laser beam scanning.** Scans an image directly toward the retina. Always-in-focus, but laser safety and alignment are serious — not a beginner path.

**Verdict:** start with magnifier-on-a-bench to validate, then build a **birdbath** for the wearable.

### Math

A near-eye display is a magnifier: a small panel sits near the focal point of an optic so the eye sees a large *virtual* image at a comfortable distance.

- **Field of view (single-lens approximation):** for a panel of height `h` behind an optic of focal length `f`,
  `FOV_vertical ≈ 2 · arctan( (h/2) / f )`.
  Same for width with the panel width.

- **Worked example.** A 0.2-inch (≈5.1 mm diagonal) 16:9 microdisplay has roughly 2.5 mm height × 4.45 mm width. With `f = 15 mm`:
  - vertical FOV ≈ 2·arctan(1.25/15) ≈ **9.5°**
  - horizontal FOV ≈ 2·arctan(2.22/15) ≈ **16.9°**
  - → about a **17–19° diagonal** image. Typical HUD scale.

- **Want a bigger image?** Use a *bigger panel* or a *shorter focal length*. Shorter `f` brings the optic closer to the eye and rapidly worsens aberrations; bigger panels cost money, power, and bulk. There's no free lunch.

- **Virtual image distance.** Place the panel *exactly* at the focal point → image at infinity (relaxed eyes, good for HUD). Place it slightly *inside* the focal length → image at a finite, nearer distance (e.g., 2 m), which many find more comfortable and easier to converge on.

- **Eyebox and eye relief.** The **eyebox** is the volume where your pupil can sit and still see the full image; **eye relief** is how far the optic is from your eye. Glasses need ~15–25 mm relief and an eyebox large enough to tolerate fit shift and your interpupillary distance (IPD). 

- **The fundamental constraint (étendue / Lagrange invariant).** You cannot simultaneously maximize FOV, eyebox, *and* compactness. Push two and the third suffers. Internalize this early — it explains every commercial compromise and will explain yours.

### Practical optics tips
- Prototype on an **optical breadboard** (or a flat board with adjustable mounts) before you 3D-print anything. Get focus, FOV, and alignment right in free space first.
- Beamsplitter **ratio** matters: a 50/50 splits image and world-light evenly; a 70/30 or 30/70 lets you bias toward image brightness or world clarity.
- Stray light and ghosting are the enemies — add baffles/flocking around the path.
- Eyeglass wearers: plan for **prescription inserts** behind the combiner rather than building the combiner into a prescription lens.

---

## 4. The display / light engine

The microdisplay is the image source. Sub-inch panels are the norm; you magnify them with the optics above.

### Technology choices
- **Micro-OLED (OLED-on-silicon / OLEDoS).** Excellent contrast and color, true blacks, very high pixel density (commercial panels reach into the **thousands of PPI**, ~4,000–5,000). The most popular and accessible choice for makers; widely available as modules with driver boards. Brightness (typically up to a few thousand nits at panel) is fine indoors and through efficient optics, but **marginal for bright outdoor see-through**.
- **LCoS (liquid crystal on silicon).** Reflective, needs an illumination source; mature, used in many AR dev kits. More optical plumbing than OLED.
- **DLP (digital micromirror).** Bright, fast; bulkier engine.
- **MicroLED.** The future for outdoor AR: self-emissive and astonishingly bright — panels can reach into the **millions of nits**, which is what waveguides' tiny throughput demands. Still expensive, mostly monochrome or early full-color, and dominated by a small number of suppliers. Overkill and over-budget for a first HUD; worth watching.

**Verdict:** use a **Micro-OLED module with an HDMI- or MIPI-DSI-input driver board** for your first build. It's the path of least resistance.

### What to look for in a module
- **Interface:** a board accepting **HDMI** is easiest to drive from an SBC or laptop; **MIPI-DSI** is lighter/more integrated but needs a host that can emit DSI (or a bridge chip). A 0.2–0.5-inch panel at 640×400 up to 1920×1080 is a sensible range.
- **Brightness** (nits at panel) and **power draw** — both feed your thermal and battery budget.
- **Driver board size** — it has to fit in a temple arm.
- Monocular = one panel + one driver. Binocular = two, plus the headache of feeding both and color/brightness matching them.

---

## 5. Compute

Decide the split (see §2), then pick silicon for each side.

### On-glasses (always present)
A **microcontroller acting as a sensor hub + power manager**: reads the IMU/sensors, does light filtering, manages the battery and charging, handles buttons/touch, and talks to the host.
- **ESP32-S3** is a strong pick: Wi-Fi + BLE built in, plenty of I²C/SPI, low power, tiny. Great for the sensor-hub role and for wireless control.
- An **nRF52/nRF53** is even more power-frugal if you mainly need BLE.

### Host (renders the UI / runs apps)
- **Tier 1 (HUD):** your **phone** is the best host. It has the GPU, the radios, the battery, and the data (maps, notifications). Glasses become a Bluetooth/USB display + sensor peripheral. Lowest weight on your face.
- **Tier 1–2 standalone:** a **Raspberry Pi Compute Module 4/5** (on a custom or off-the-shelf carrier) gives real GPU and Linux in a small footprint. A **Raspberry Pi Zero 2 W** is lighter for very simple overlays.
- **Tier 2–3 with vision/SLAM:** an **NVIDIA Jetson** (Orin Nano class) provides the GPU/AI throughput for computer vision, but it's hot and power-hungry — pocket it, don't face-mount it.

**Don't** try to render world-locked AR on a bare MCU. MCUs run the sensors and the plumbing; a GPU-class device renders the scene.

---

## 6. Sensors

Match the sensor suite to your tier — don't pay the compute/power cost of sensors your UX doesn't use.

- **IMU (9-DoF: accel + gyro + magnetometer).** *Mandatory.* Gives head orientation at high rate for low-latency, stable overlays. The gyro is fast but drifts; the accelerometer and magnetometer correct drift via sensor fusion (a Madgwick/Mahony or Kalman filter). For a head-locked HUD, a good fused IMU is *all* you need.
- **Camera (RGB, CSI/MIPI).** Needed for any computer vision: object/text recognition, QR/marker tracking, photo capture, and — combined with the IMU — **visual-inertial odometry (VIO)**, the backbone of positional tracking.
- **Depth / Time-of-Flight (ToF) or stereo cameras.** Only for Tier 3: building a 3D mesh of the room so virtual objects can sit on surfaces and **occlude** correctly. Heavy compute.
- **Ambient light sensor (ALS).** Cheap and worth it: auto-dims the display so the overlay is readable indoors and not blinding at night, and saves power.
- **Microphone(s).** For voice commands / assistant input; an array enables beamforming.
- **Eye-tracking (IR cameras).** Advanced. Enables **foveated rendering** (full detail only where you look, saving GPU) and gaze-based interaction. Skip for a first build.

**Tier 1 sensor kit:** 9-DoF IMU + ALS + a button/touch strip. That's it.

---

## 7. Connectivity

- **USB-C** — the workhorse for a tethered build: carries video (DisplayPort Alt Mode), data, *and* power in one cable. Strongly preferred for Tier 1–2 prototypes.
- **BLE (Bluetooth Low Energy)** — low-power control channel and for streaming sensor data / notifications from a phone. Ideal for a wireless HUD where the phone does the heavy lifting.
- **Wi-Fi** — higher bandwidth (e.g., streaming rendered frames or camera video) at a real power cost.

A clean Tier 1 wireless design: phone ↔ glasses over **BLE** for control + notification payloads, glasses render simple overlays locally on the MCU/SBC.

---

## 8. Power and thermals

Power is a constant fight because the battery sits on your head (or you tether to move it).

### Build a power budget (example, monocular HUD)
| Subsystem | Typical draw |
|---|---|
| Micro-OLED panel + driver | 0.3–1.0 W |
| SBC host (if on-glasses, e.g. Pi Zero 2 W) | 1–3 W |
| MCU sensor hub (ESP32-S3) | 0.1–0.5 W |
| IMU + ALS + misc sensors | <0.2 W |
| Radios (BLE idle→active) | 0.05–0.5 W |
| **Total (on-glasses compute)** | **~2–5 W** |
| **Total (phone-tethered, glasses dumb)** | **~0.7–2 W** |

### Battery sizing
Energy (Wh) = battery capacity. A small **LiPo** of ~500 mAh @ 3.7 V ≈ 1.85 Wh. At 2 W that's under an hour; at 1 W (tethered/dumb-glasses) ~1.5–2 h. This is exactly why HUDs tether or keep on-glasses compute minimal.

### Power essentials (do not skip)
- Use a proper **battery management system (BMS)** / protected LiPo: over-charge, over-discharge, and short protection. Lithium cells are unforgiving.
- Add **USB-C PD** charging and a clean buck/boost rail for your logic voltages.
- **Thermals:** anything on your temple that runs warm is uncomfortable fast. Keep hot silicon (SBC/Jetson) off the face — pocket it and tether — or budget for spreading heat across metal frame elements. A face-mounted >2–3 W hotspot is a comfort failure.
- Distribute battery weight toward the back of the temples / behind the ears to balance the head's center of gravity.

---

## 9. Mechanical / frame design

The frame holds optics, electronics, and battery in precise alignment and has to be wearable.

- **3D printing** is the maker's core tool. Design in CAD (Fusion 360, FreeCAD, Onshape). Print in lightweight, slightly flexible materials — **nylon (PA12/SLS if you can), PETG, or resin** for fine optical mounts.
- **Optical mounts are the precision-critical parts.** The display, lens, beamsplitter, and mirror must hold position to sub-millimeter and sub-degree tolerances, or the image blurs, ghosts, or drifts off the eyebox. Design **adjustable** mounts (slotted holes, set screws, shims) for the first frame; once aligned, you can print a fixed version.
- **Center of gravity** as close to the head as possible; pad the **nose bridge** and **temples**; aim for even weight distribution to avoid pressure points and neck strain.
- Plan **cable/flex routing** through the temples; leave service access — you *will* re-open this many times.
- Iterate: expect several frame revisions. Early "frames" can be ugly bench fixtures; refine toward wearable once optics and electronics are locked.

---

## 10. The software stack and the UX overlay

 the layers that turn sensor data and app logic into a stable, readable interface in front of the eye.

### The layers (bottom to top)
1. **Firmware (on the MCU):** sensor drivers, IMU sampling, the **sensor-fusion filter** (Madgwick/Mahony/EKF) that turns raw accel/gyro/mag into a clean orientation quaternion, power management, and the comms protocol to the host. Run fusion at a high, fixed rate (e.g., 100–1000 Hz).
2. **Host runtime / OS:** Linux on an SBC, or an app on the phone. Receives orientation + inputs, runs app logic, and owns rendering.
3. **Rendering / compositor:** draws the UI and composites it for the display. Options:
   - Lightweight 2D HUDs: a graphics library / game framework, or even direct framebuffer drawing.
   - 3D / spatial: a **game engine** (Unity, Unreal, Godot) or a graphics API (OpenGL ES, Vulkan, WebGL/Three.js for web-based glasses).
   - For spatial AR, target **OpenXR** — a vendor-neutral AR/VR API — so your app logic isn't welded to one device.
4. **Tracking / world model:** for head-locked, just consume the fused IMU orientation. For world-locked (Tier 3), a **SLAM/VIO** pipeline (e.g., built on ORB-SLAM-style or OpenVINS-style approaches) fuses camera + IMU to estimate 6-DoF pose, plus mapping for occlusion.
5. **The overlay UI itself** (the UX).

### Designing the overlay UX (this is what users feel)
Optical see-through HUD UX has hard rules that differ from a phone screen:

- **It's additive, so design for it.** Light *adds* to the world — you can't display true black (black = transparent), and bright backgrounds wash out the image. Use **light text/icons on dark or no fill**, high contrast, and bold strokes. Avoid large filled panels.
- **Glanceable, peripheral, minimal.** The overlay competes with the real world for attention. Keep it sparse: a few widgets, large legible type, information that's *glanceable in under a second*. Park persistent UI toward the **periphery**, leave the central view clear.
- **Latency is a UX feature, not a nicety.** For head-locked content the budget is loose, but anything that lags head motion feels broken and can cause discomfort. Keep **motion-to-photon latency low** — fast IMU, fast fusion, fast render, fast panel. For world-locked content, latency *is* the difference between "anchored" and "swimming."
- **Stability over richness.** A rock-steady simple overlay beats a jittery fancy one every time. Filter/smooth, but not so much you add lag.
- **Auto-brightness.** Tie display brightness to the ALS so the UI is readable indoors and not blinding at night.
- **Safety-aware:** don't occlude the user's path or critical sightlines; fade or hide UI when motion suggests walking/driving.

### Interaction model
Glasses have no keyboard. Pick one or two and design around them:
- **Touch strip / buttons on the temple** — reliable, private, simple. Great default.
- **Voice** — natural for assistant-style input; needs mic + wake-word/ASR (often offloaded to the phone/cloud).
- **Head gestures** (nod/tilt) — cheap with the IMU you already have.
- **Hand gestures** — needs camera + CV; powerful but heavy.
- **Gaze** — needs eye-tracking; advanced.

### Calibration (don't skip)
Build a **calibration routine**: align the rendered image to the user's eye (position/scale/IPD), set the virtual image distance, and — for world-locked — calibrate camera–IMU extrinsics and display-to-eye mapping. A per-user fit calibration is what makes the overlay feel "right."

---

## 11. Build Plans Future

1. **Optics on the bench.** Microdisplay + lens + beamsplitter/mirror on a breadboard. Drive the panel from a laptop over HDMI. Goal: a focused, correctly-sized virtual image you can see with your eye at the right relief. Measure your real FOV and eyebox. 
2. **Sensors + fusion on the bench.** Wire the IMU to the ESP32-S3, run the fusion filter, visualize orientation on a PC (spin a cube). Tune until stable and drift-free.
3. **Render an overlay on the host.** Build the UI (start with a clock + a notification + a heading indicator). Feed it the live orientation so the overlay reacts to head motion. Tune latency and smoothing.
4. **Close the loop tethered.** Combine: host renders → panel displays; MCU streams sensor data → host. Confirm the overlay is stable, readable, and low-latency *through the optics*, on your bench rig.
5. **Power + integration.** Add the LiPo/BMS/PD, the ALS auto-brightness, the touch/voice input. Validate runtime and that nothing gets uncomfortably warm.
6. **Mechanical integration.** Move from breadboard to a 3D-printed frame with adjustable optical mounts. Re-align. Iterate the print for fit, weight, and balance.
7. **Polish.** Calibration routine, UX refinement, robustness, then a fixed (non-adjustable) frame revision.
8. **(Only if going Tier 3)** add camera(s)/ToF and layer in VIO/SLAM and occlusion — a major project of its own.

---

## 12. Bill of materials 

**Tier 1 — monocular HUD, phone-tethered (~$250–$600)**
- Micro-OLED module + HDMI/MIPI driver board
- Beamsplitter (plate or cube) + small concave mirror, or a birdbath kit
- Collimating lens (+ assorted lenses for tuning)
- 9-DoF IMU breakout (e.g., a BNO-class fused IMU simplifies firmware)
- ESP32-S3 dev board
- Ambient light sensor, capacitive touch strip / buttons
- Small LiPo + protected charger/PD board
- USB-C cable/connectors, wiring, perfboard/custom PCB
- 3D-printing filament/resin, fasteners, optical flocking/baffle material

**Tier 2 — binocular, on-glasses SBC (add ~$200–$500)**
- Second display + driver; matched pair
- Raspberry Pi CM4/CM5 + carrier (or Pi Zero 2 W for simple overlays)
- Larger battery, better thermal spreading
- Possibly a sourced **waveguide evaluation module** (significant cost) instead of birdbath

**Tier 3 — spatial AR (substantial)**
- Camera module(s) (CSI), optional ToF/stereo
- Jetson Orin Nano-class compute (pocketed)
- The real cost is *time*: SLAM, calibration, occlusion.

*Source Prices Changing*

---

## 13. Safety

- **Eyes/optics:** keep brightness comfortable; auto-dim. **Never** build a retinal-laser-scanning design without proper laser-safety knowledge and eye-safe power limits — this can cause permanent damage.
- **Battery:** use protected cells/BMS, correct charging, and don't enclose LiPos where a fault can't vent. Lithium fires are real.
- **Situational awareness:** the overlay must not block the user's view of the world during movement. Design fail-safe (UI fades/hides) and never obstruct central vision or path.
- **Privacy:** if you add a camera, add a visible recording indicator and be mindful of where/when it's used.

---

## 14. Common pitfalls 

- **Skipping the bench optics stage** and 3D-printing a frame around optics you haven't validated. Always free-space-align first.
- **Chasing FOV.** Big FOV fights eyebox and form factor (étendue). Modest FOV, done well, beats ambitious FOV done badly.
- **Underestimating brightness** for outdoor use — birdbath + Micro-OLED is an *indoor* combo unless you invest heavily.
- **Putting hot, hungry compute on the face.** 
- **Ignoring latency and stability** in the UX — a laggy or jittery overlay feels broken regardless of how pretty it is.
- **Rigid optical mounts on the first frame** — you need adjustment to align, then lock it in.
- **Trying to render world-locked AR on an MCU** — wrong tool; you need a GPU-class host and a real tracking pipeline.

---

### TL;DR
Build a **monocular, optical see-through, birdbath HUD**, drive a **Micro-OLED panel over HDMI**, render the overlay on a **phone or small SBC**, get head orientation from a **fused 9-DoF IMU on an ESP32-S3**, and obsess over a **stable, glanceable, high-contrast, low-latency overlay**. Prove the optics on a bench first, integrate in stages, and only climb toward binocular waveguides and world-locked spatial AR once Tier 1 actually works on your face.
