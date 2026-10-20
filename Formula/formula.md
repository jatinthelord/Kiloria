# KiloriaT: the math formulas your engine really uses

I copied every formula here from your code, using your own constants and variable names. Each one has a line number so you can find it.
**Two files matter most:**
- `MEC` means `src/ModernEngineCore.cpp`. Your editor compiles the GLSL that is written inside this file (lines 391–481).
- `frag` and `vert` mean `shaders/studio_pbr.frag` and `shaders/studio_pbr.vert`, which are copies of the same shader code.

Some tiny helpers show up again and again:

$$
\mathrm{mix}(a,b,t)=a+(b-a)\,t \qquad \mathrm{clamp}(x,lo,hi)=\min(\max(x,lo),hi)
$$

$$
\mathrm{smoothstep}(e_0,e_1,x)=t^2(3-2t),\quad t=\mathrm{clamp}\!\left(\tfrac{x-e_0}{e_1-e_0},0,1\right)
$$

- **mix:** slides from \(a\) to \(b\).
- **clamp:** keeps a number inside a range.
- **smoothstep:** makes a soft S-shaped change from 0 to 1 instead of a sudden jump.

---

## 1. Transforms & Camera

### 1.1 Model matrix: where an object sits in the world
$$
M = T(\text{location})\;R_z(\theta_z)\;R_y(\theta_y)\;R_x(\theta_x)\;S(\text{scale})
$$
You read this right to left: first scale the object, then spin it around X, Y and Z, then move it into place.
- \(T\) is translation, \(R\) is rotation, \(S\) is scale.
- **Source:** `MEC:894-895`

### 1.2 Rotation matrices
$$
R_y(a)=\begin{pmatrix}\cos a&0&\sin a&0\\0&1&0&0\\-\sin a&0&\cos a&0\\0&0&0&1\end{pmatrix}
$$
This spins a point around the Y axis (the up axis). \(R_x\) and \(R_z\) follow the same pattern on the other two axes.
- \(a\) is the angle in radians.
- **Source:** `MEC:766-777`

### 1.3 Quaternion to rotation matrix
$$
R(q)=\begin{pmatrix}
1-2(y^2+z^2) & 2(xy-wz) & 2(xz+wy)\\
2(xy+wz) & 1-2(x^2+z^2) & 2(yz-wx)\\
2(xz-wy) & 2(yz+wx) & 1-2(x^2+y^2)
\end{pmatrix}
$$
A quaternion is a 4-number way to store a rotation, and this turns it into a normal 3×3 rotation matrix.
- \(q=(x,y,z,w)\), normalized first.
- **Source:** `include/kiloria/math/mat4.hpp:38-54`

### 1.4 Perspective projection: things far away look smaller
$$
s=\frac{1}{\tan(\text{fov}/2)},\qquad
P=\begin{pmatrix}
\frac{s}{\text{aspect}}&0&0&0\\
0&s&0&0\\
0&0&-\frac{f+n}{f-n}&-\frac{2fn}{f-n}\\
0&0&-1&0
\end{pmatrix}
$$
This squeezes the 3D world into the screen so that far things shrink.
- \(\text{fov}=0.785398163\) rad, which is 45°.
- \(\text{aspect}\) is width ÷ height.
- \(n=0.05\) is the near plane and \(f=1000\) is the far plane.
- **Source:** `mat4.hpp:55-69`, called at `MEC:849`

### 1.5 LookAt (view matrix): building the camera from 3 vectors
$$
\vec f=\frac{\text{target}-\text{eye}}{\|\text{target}-\text{eye}\|},\quad
\vec s=\frac{\vec f\times\vec{up}}{\|\vec f\times\vec{up}\|},\quad
\vec u=\vec s\times\vec f
$$
$$
V=\begin{pmatrix}s_x&s_y&s_z&-\vec s\cdot\text{eye}\\u_x&u_y&u_z&-\vec u\cdot\text{eye}\\-f_x&-f_y&-f_z&\vec f\cdot\text{eye}\\0&0&0&1\end{pmatrix}
$$
This uses your cross product and normalize lessons. The camera works out its forward, right and up directions, then moves the world so the camera sits at the center.
- \(\vec f\) is forward, \(\vec s\) is side (right), \(\vec u\) is the corrected up.
- **Source:** `mat4.hpp:70-84`

### 1.6 The full vertex pipeline
$$
\text{gl\_Position}=P\,V\,M\,\begin{pmatrix}p\\1\end{pmatrix}
$$
Every corner of every mesh goes through 3 matrix multiplies: object space to world space to camera space to screen.
- \(p\) is the vertex position.
- **Source:** `MEC:397-398`, `vert:18-21`

### 1.7 Normal matrix
$$
N=\mathrm{normalize}\!\left((M_{3\times3}^{-1})^{T}\,n\right)
$$
Normals (the "which way is the surface facing" arrows) need the inverse-transpose matrix so they stay correct when you stretch an object.
- **Source:** `MEC:398`, `vert:20`

### 1.8 Perspective divide: world point to screen pixel
$$
\text{ndc}=\frac{(x,y,z)}{w},\qquad
\text{pixel}_x=o_x+\frac{\text{ndc}_x+1}{2}W,\quad
\text{pixel}_y=o_y+\frac{1-\text{ndc}_y}{2}H
$$
Dividing by \(w\) is where the "far = small" effect actually happens. The second part turns −1…1 into pixel positions for your gizmo.
- **Source:** `mat4.hpp:93-100`, `MEC:1243-1245`

---

## 2. Lighting / PBR (physically based rendering)

### 2.1 GGX normal distribution \(D\): how shiny the highlight is
$$
\alpha=r^2,\quad d=(\vec n\cdot\vec h)^2(\alpha^2-1)+1,\quad D=\frac{\alpha^2}{\pi d^2}
$$
Rough surfaces get a wide, soft highlight and smooth ones get a tiny sharp dot.
- \(r\) is roughness, clamped to between 0.045 and 1.
- \(\vec h=\mathrm{normalize}(\vec v+\vec l)\) is the half vector.
- **Source:** `MEC:406-407`, `frag:43-49`

### 2.2 Smith geometry term \(G\): tiny bumps shadowing each other
$$
k=\frac{(\alpha+1)^2}{8},\qquad G_1(x)=\frac{x}{x(1-k)+k},\qquad G=G_1(\vec n\cdot\vec v)\,G_1(\vec n\cdot\vec l)
$$
Microscopic bumps on a rough surface block some light. \(G\) tells you how much gets through.
- **Source:** `MEC:408-409`, `frag:51-55`

### 2.3 Fresnel-Schlick \(F\): surfaces get shiny at grazing angles
$$
F=F_0+(1-F_0)\,(1-\vec h\cdot\vec v)^5,\qquad F_0=\mathrm{mix}(0.04,\ \text{albedo},\ \text{metal})
$$
Look at a table from the side and it reflects more. This formula is the reason.
- \(F_0\) is the reflectivity when you look straight on: 0.04 for plastic or stone, the object's own color for metal.
- **Source:** `MEC:410,417`, `frag:57-59,83`

### 2.4 Cook-Torrance BRDF: the main lighting formula
$$
\text{spec}=\frac{D\,G\,F}{4(\vec n\cdot\vec v)(\vec n\cdot\vec l)},\qquad
\text{diff}=(1-F)(1-\text{metal})\frac{\text{albedo}}{\pi}
$$
$$
L_{\text{sun}}=(\text{diff}+\text{spec})\cdot(\text{sunColor}\cdot\text{sunIntensity})\cdot\max(\vec n\cdot\vec l,0)
$$
The light you see is diffuse (the matte color) plus specular (the shiny part), times how hard the sun hits the surface (your dot product).
- \(\vec l\) points toward the sun and \(\vec v\) points toward the camera.
- **Source:** `MEC:415-420,430`, `frag:78-90,111`

### 2.5 Sky-fill (ambient) light
$$
\text{hdr}=\text{surface}\cdot\text{ambientIntensity}\cdot(0.45+0.55\max(n_y,0))
$$
Surfaces that face up get more sky light than walls do, so even shadows aren't pitch black.
- **Source:** `MEC:429`, `frag:110`

---

## 3. Sky, sun, moon & day/night

### 3.1 Day-night cycle (600-second day)
$$
t\leftarrow (t+\min(\Delta t,0.1)) \bmod 600,\qquad \theta=t\cdot\frac{2\pi}{600}
$$
$$
\vec{sun}=\mathrm{normalize}(0.98\cos\theta,\ \sin\theta,\ 0.20)
$$
The sun travels around a full circle every 10 minutes, and \(\sin\theta<0\) means night.
- **Source:** `MEC:783-784,791`

### 3.2 Sun from the sliders (azimuth & elevation)
$$
\vec{sun}=\mathrm{normalize}\big(\cos\phi\cos|e|,\ \sin|e|,\ \sin\phi\cos|e|\big)
$$
These are spherical coordinates: \(\phi\) turns you left or right and \(e\) tilts you up or down.
- **Source:** `MEC:786-787`

### 3.3 Daylight factor
$$
d=\mathrm{smoothstep}(-0.13,\ 0.16,\ \text{sun}_y)
$$
$$
\text{sunIntensity}'=\text{sunIntensity}\cdot d,\qquad \text{ambient}'=\text{ambient}\cdot(0.28+0.72\,d)
$$
This fades the sun out when it goes below the horizon, but keeps 28% of the sky light at night.
- **Source:** `MEC:872-876` (CPU), `MEC:463` (sky shader)

### 3.4 Camera ray for every sky pixel
$$
\vec{ray}=\mathrm{normalize}\big(\vec F+\vec R\,(x_{ndc}\cdot\text{aspect}\cdot\tan(\text{fov}/2))+\vec U\,(y_{ndc}\tan(\text{fov}/2))\big)
$$
Each pixel works out which direction it is looking, so the sky knows what color to be there.
- **Source:** `MEC:461,808`

### 3.5 Sky gradient & sunset
$$
\text{sky}=\mathrm{mix}\big(\text{horizon},\ \text{zenith},\ \mathrm{smoothstep}(0.04,0.92,\ 0.5\,ray_y+0.5)\big)
$$
$$
\text{twilight}=1-\mathrm{smoothstep}(0.02,0.38,|\text{sun}_y|),\qquad
\text{sky}=\mathrm{mix}\big(\text{sky},\ (1,.35,.11),\ 0.30\cdot\text{twilight}\cdot e^{-3\max(ray_y,0)}\big)
$$
The sky is lighter blue at the horizon and darker overhead. Orange shows up near the horizon only when the sun is low.
- **Source:** `MEC:463-470`

### 3.6 Sun disk + glow (Gaussian)
$$
\gamma=\arccos(\vec{ray}\cdot\vec{sun}),\quad
\text{disk}=1-\mathrm{smoothstep}(0.006,0.010,\gamma)
$$
$$
\text{glow}=0.55\,e^{-\gamma^2/0.055^2}+0.12\,e^{-\gamma^2/0.20^2}
$$
\(\gamma\) is the angle between where you look and the sun. A small angle draws the disk, and a bell curve (\(e^{-x^2}\)) draws the soft glow around it.
- **Source:** `MEC:471-474`

### 3.7 Moon and stars
$$
\text{moonDir}=-\vec{sun},\qquad \text{moonGlow}=(.11,.17,.29)\,e^{-\gamma_m^2/0.045^2}\cdot(1-d)
$$
$$
\text{star}=\mathrm{step}\big(0.997,\ \text{hash}(\lfloor 240\,\vec{ray}\rfloor)\big)\cdot(1-d)\cdot\mathrm{smoothstep}(0.1,0.55,ray_y)
$$
The moon is always opposite the sun. The sky is cut into tiny cells, and about 0.3% of them become stars.
- **Source:** `MEC:475-480`

> There's no fog in this snapshot. I searched for `fog` and nothing matched.

---

## 4. Tonemapping & color

### 4.1 Exposure tonemap
$$
\text{mapped}=1-e^{-\text{hdr}\cdot\text{exposure}}
$$
Bright light can be any size (HDR), and this squishes it into 0…1 so the screen can show it.
- **Source:** `MEC:434`, `frag:120`

### 4.2 Gamma correction
$$
\text{color}=\text{mapped}^{1/2.2}
$$
Monitors darken colors, so you brighten them in advance to cancel it out.
- **Source:** `MEC:437`, `frag:124`

### 4.3 Anti-aliased editor grid
$$
c=\frac{|\mathrm{fract}(xz-0.5)-0.5|}{\mathrm{fwidth}(xz)},\qquad \text{line}=1-\min(c_x,c_y,1)
$$
This draws grid lines exactly 1 pixel wide at any distance, because `fwidth` measures how big a pixel is in world units.
- **Source:** `MEC:435-436`, `frag:121-123`

---

## 5. Water

### 5.1 Wave height (vertex shader)
$$
y \mathrel{+}= 0.025\,\sin(2.1x+1.7z+1.4t)
$$
This moves the top of the water up and down in a sine wave.
- **Source:** `MEC:396`, `vert:17`

### 5.2 Ripple color + Fresnel
$$
\text{ripple}=0.5+0.5\sin(2.2x+1.7z-0.8t)
$$
$$
\text{water}=(.012,.10,.18)+\text{ripple}\cdot(.025,.07,.10)
$$
$$
F_w=(1-\max(\vec n\cdot\vec v,0))^5,\qquad
\text{hdr}=\mathrm{mix}\big(\text{hdr},\ \text{water}+(.25,.36,.43)F_w,\ 0.80\big)
$$
The water gets darker and lighter blue stripes, and it looks brighter when you see it from a low angle (Fresnel). Then the final pixel is drawn 76% opaque.
- **Source:** `MEC:431-433,437`, `frag:113-118`

---

## 6. Buoyancy & simple rigid bodies (what Play mode really uses)

### 6.1 How much of a box is underwater
$$
s=\mathrm{clamp}\!\left(\frac{y_{\text{water}}-(y-h)}{2h},\ 0,\ 1\right)
$$
This is the fraction of the box below the water, from 0 (dry) to 1 (fully under).
- \(h\) is half the box height and \(y-h\) is the bottom of the box.
- **Source:** `MEC:1575`

### 6.2 Buoyancy force (Archimedes)
$$
a_y=-9.81+9.81\cdot1.42\cdot s
$$
Gravity pulls down and water pushes up harder the deeper the box goes. Fun fact: it floats when \(s=1/1.42\approx0.70\), so boxes float with about 70% of their height underwater.
- **Source:** `MEC:1570,1576`

### 6.3 Water drag (exponential damping)
$$
v_{x,z}\leftarrow v_{x,z}\,e^{-1.8\,s\,\Delta t},\qquad v_y\leftarrow v_y\,e^{-2.4\,s\,\Delta t},\qquad \omega\leftarrow\omega\,e^{-0.32\,\Delta t}
$$
Water slows things down smoothly, and using \(e^{-k\Delta t}\) means the result is the same at any frame rate.
- **Source:** `MEC:1577-1579,1603`

### 6.4 Semi-implicit Euler (moving things each frame)
$$
v_y\leftarrow v_y+a_y\Delta t,\qquad y\leftarrow y+v_y\Delta t,\qquad \Delta t=\mathrm{clamp}(\text{frame},0,0.05)
$$
Update the speed first, then use the new speed to update the position. This keeps the motion stable.
- **Source:** `MEC:1558,1581-1582`

---

## 7. Player & camera movement

### 7.1 Look direction from yaw & pitch
$$
\vec{fwd}=(\sin\psi\cos\varphi,\ \sin\varphi,\ -\cos\psi\cos\varphi)
$$
Two angles, left-right (\(\psi\), yaw) and up-down (\(\varphi\), pitch), make a unit-length 3D direction vector.
- **Source:** `MEC:703-705,823-824`

### 7.2 Mouse look
$$
\psi\leftarrow\psi-0.0018\,\Delta m_x,\qquad \varphi\leftarrow\mathrm{clamp}(\varphi\mp0.0018\,\Delta m_y,\ -1.48,\ 1.48)
$$
Moving the mouse turns the camera, and the pitch limit stops you from flipping upside down.
- **Source:** `MEC:243,1329-1331`

### 7.3 Walking
$$
\vec{dir}=\mathrm{normalize}(\pm\vec{forward}\pm\vec{right}),\quad
\vec{forward}=(\sin\psi,0,-\cos\psi),\ \vec{right}=(\cos\psi,0,\sin\psi)
$$
$$
\text{speed}=4.5\times\{1,\ 0.52\text{ crouch},\ 0.22\text{ crawl}\},\qquad x\leftarrow x+dir_x\cdot\text{speed}\cdot\Delta t
$$
Normalizing means walking diagonally isn't faster than walking straight. That's your normalize lesson at work.
- **Source:** `MEC:130,1606-1619`

### 7.4 Jump & gravity
$$
v_y=5.0\ \text{(on jump)},\qquad v_y\leftarrow v_y-9.81\Delta t,\qquad y\leftarrow\max(y+v_y\Delta t,\ \text{terrainHeight})
$$
You jump up at 5 m/s and gravity pulls you back down. Fun fact: the highest point is \(v^2/2g=25/19.62\approx1.27\) m.
- **Source:** `MEC:131,1622-1634`

### 7.5 Smooth third-person camera (spring)
$$
\alpha=1-e^{-8.5\,\Delta t},\qquad \text{cam}\leftarrow\text{cam}+(\text{desired}-\text{cam})\,\alpha
$$
$$
\text{desired}=\text{player}-4.2\,\vec{fwd}+(0,2.15,0)
$$
Each frame the camera moves part of the way toward where it wants to be, so it follows you smoothly instead of snapping.
- **Source:** `MEC:831-836`

### 7.6 Editor orbit camera
$$
\text{eye}=(D\cos p\sin y,\ D\sin p+1,\ D\cos p\cos y),\qquad D\in[2,80]
$$
The camera rides on a ball of radius \(D\) around the point \((0,0.4,0)\).
- **Source:** `MEC:839-842,1290-1293`

### 7.7 Pick-up ray vs box (slab test)
$$
t_0=\frac{lo_i-o_i}{d_i},\ t_1=\frac{hi_i-o_i}{d_i},\quad t_{near}=\max_i\min(t_0,t_1),\ t_{far}=\min_i\max(t_0,t_1)
$$
This shoots a laser from your eyes. It hits the box if \(t_{near}\le t_{far}\) and the box is within 5.5 m.
- **Source:** `MEC:735-744`

### 7.8 Throwing
$$
\vec v=10.5\,\vec{fwd}+(0,2.5,0)
$$
A thrown object flies forward at 10.5 m/s with a small push upward.
- **Source:** `MEC:752`

---

## 8. Particles

### 8.1 Particle motion & fade
$$
v_y\leftarrow v_y-0.18\Delta t,\quad \vec p\leftarrow\vec p+\vec v\Delta t,\quad \text{alpha}=\text{alpha}_0\cdot\mathrm{clamp}\!\left(\frac{\text{life}}{\text{maxLife}},0,1\right)
$$
Particles get a tiny bit of gravity (0.18) and fade out as their life runs down.
- Splash particles live 0.9 s and gas particles live 2.8 s.
- **Source:** `MEC:665,714,932`

### 8.2 Pseudo-random spread
$$
a=\sin(12.9898\,\text{seed}),\ b=\cos(7.233\,\text{seed}),\quad \vec v=(2a,\ 1.3,\ 2b)\ \text{(splash)}
$$
Sine of a big number jumps around, so it works as a cheap "random" number.
- **Source:** `MEC:712-713`

### 8.3 Point size by distance
$$
\text{size}_{px}=\mathrm{clamp}\!\left(\frac{720\cdot\text{size}}{\max(-z_{view},0.15)},\ 2,\ 72\right)
$$
Size divided by distance makes far particles look smaller, which is perspective again.
- **Source:** `MEC:443`

### 8.4 Soft round particle
$$
r^2=\vec q\cdot\vec q,\quad \text{edge}=1-\mathrm{smoothstep}(0.42,1.0,r^2)
$$
This turns a square dot into a soft circle, using the dot product of a point with itself (the squared length).
- **Source:** `MEC:447-448`

### 8.5 Cone emitter (library)
$$
\vec v=\text{speed}\,(\vec{axis}\cos\theta+\vec{radial}\sin\theta),\quad \vec{radial}=\vec t\cos\phi+\vec b\sin\phi
$$
This shoots particles in a cone. The tangent \(\vec t\) and bitangent \(\vec b\) come from cross products with the axis.
- **Source:** `src/particles.cpp:25-32`

---

## 9. Terrain & noise

### 9.1 Perlin fade curve
$$
\text{fade}(t)=6t^5-15t^4+10t^3
$$
This blends smoothly between grid points so the hills don't have sharp creases.
- **Source:** `src/procedural_terrain.cpp:35-37`

### 9.2 Perlin noise sample
$$
n=\sqrt2\cdot\mathrm{lerp}\big(\mathrm{lerp}(n_{00},n_{10},u),\ \mathrm{lerp}(n_{01},n_{11},u),\ v\big)
$$
Each grid corner has a random arrow. You take a dot product with each arrow and blend the four results.
- **Source:** `procedural_terrain.cpp:54-68`

### 9.3 Fractal noise (fBm): big hills + small bumps
$$
h=2.4\cdot\frac{\sum_{k=0}^{4}0.48^k\ \text{noise}(0.035\cdot2^k\,x,\ 0.035\cdot2^k\,z)}{\sum_{k=0}^{4}0.48^k}
$$
This adds 5 layers of noise. Each layer has twice the detail and is 0.48 times as strong as the one before.
- **Source:** `procedural_terrain.cpp:70-83,100-101`, settings at `MEC:189-192`

### 9.4 Bilinear height lookup
$$
h(x,z)=\mathrm{lerp}\big(\mathrm{lerp}(h_{00},h_{10},t_x),\ \mathrm{lerp}(h_{01},h_{11},t_x),\ t_z\big)
$$
This finds the ground height between grid points. The player and the boxes stand on it.
- **Source:** `src/heightfield.cpp:105-119`

### 9.5 Terrain normal from slopes
$$
\vec n=\mathrm{normalize}\!\left(-\frac{\partial h}{\partial x},\ 1,\ -\frac{\partial h}{\partial z}\right)
$$
The steeper the hill, the more the normal arrow leans over. This is a first peek at calculus.
- **Source:** `heightfield.cpp:121-131`

### 9.6 Sculpt brush falloff
$$
w=(1-t)^2(1+2t),\quad t=\mathrm{clamp}(d/R,0,1)
$$
The brush is strongest in the middle and fades to 0 at its edge.
- **Source:** `heightfield.cpp:20-24`

### 9.7 Grass / rock / snow shading
$$
\text{slope}=1-\mathrm{smoothstep}(0.54,0.88,n_y),\quad \text{alt}=\mathrm{smoothstep}(3.2,5.4,y)
$$
$$
\text{surface}=\mathrm{mix}\big(\mathrm{mix}(\text{grass},\text{rock},\text{slope}),\ (.72,.75,.78),\ \text{alt}(1-0.55\,\text{slope})\big)
$$
Flat ground is grass, steep ground is rock, and high ground is supposed to be snow.
- **Source:** `MEC:423-428`, `frag:100-107`

### 9.8 Value noise (shader)
$$
\text{hash}(p)=\mathrm{fract}((p_x+p_y)p_z)\ \text{after}\ p=\mathrm{fract}(0.1031p),\ p\mathrel{+}=p\cdot(p_{yxz}+33.33)
$$
$$
f\leftarrow f^2(3-2f)
$$
This is a cheap random number per cell, smoothly blended across 8 corners. It makes the grass and rock grain.
- **Source:** `MEC:411-414`, `frag:61-76`

---

## 10. XPBD physics library (`src/xpbd_world.cpp`)

> Play mode in this snapshot doesn't call XPBD. It uses the simple physics in section 6. XPBD sits ready in the library.

### 10.1 Fixed substeps
$$
N=\left\lceil\frac{\min(\Delta t,0.25)}{1/120}\right\rceil,\qquad h=\frac{\min(\Delta t,0.25)}{N}
$$
Each frame is cut into small, equal steps of about 1/120 s so the physics stays stable.
- **Source:** `xpbd_world.cpp:157-163`

### 10.2 Predict
$$
\vec v\leftarrow\vec v+(\vec g+\vec F\,w)h,\qquad \vec x\leftarrow\vec x+\vec v h
$$
First guess where everything will move to.
- \(w=1/m\) is the inverse mass.
- **Source:** `xpbd_world.cpp:180-187`

### 10.3 XPBD distance constraint (the heart of XPBD)
$$
C=\|\vec x_b-\vec x_a\|-L,\qquad \tilde\alpha=\frac{\alpha}{h^2}
$$
$$
\Delta\lambda=\frac{-C-\tilde\alpha\lambda}{w_a+w_b+\tilde\alpha},\qquad
\vec x_a\mathrel{-}=w_a\Delta\lambda\,\hat n,\quad \vec x_b\mathrel{+}=w_b\Delta\lambda\,\hat n
$$
If two points are too far apart or too close, push them back toward the right distance \(L\). Lighter objects move more.
- \(\alpha\) is compliance (softness): 0 means a stiff rod.
- \(\hat n\) is the unit direction from a to b.
- The solver runs 8 iterations.
- **Source:** `xpbd_world.cpp:192-195,213-229`

### 10.4 Velocity from positions
$$
\vec v=\frac{\vec x-\vec x_{prev}}{h}\cdot\max(0,1-\text{damping}\cdot h)
$$
After fixing the positions, the speed is just how far each point moved divided by the time.
- **Source:** `xpbd_world.cpp:205-206`

### 10.5 Bounce impulse & material mixing
$$
j=\frac{-(1+e)\,(\vec v_{rel}\cdot\hat n)}{w_a+w_b},\qquad e=\sqrt{e_ae_b},\quad \mu=\sqrt{\mu_a\mu_b}
$$
When two balls hit, they get a kick \(j\) along the hit direction. Bounciness \(e\) and grip \(\mu\) of the two materials are mixed with a geometric mean.
- **Source:** `xpbd_world.cpp:30-37,318`

---

## 11. Formulas that look buggy

1. **Water glows at night.** This is the bug you asked about.
   - **Where:** `MEC:433`, `frag:117`:
     $$
     \text{hdr}=\mathrm{mix}(\text{hdr},\ \text{water}+(.25,.36,.43)F_w,\ 0.80)
     $$
   - **Why:** the water color is a fixed number that never gets multiplied by `sunIntensity` or `ambientIntensity`. At night everything else drops to almost 0, but 80% of the water pixel is still that fixed blue. After tonemap and gamma it comes out about 0.4 bright, while the ground is about 0.1, so the water looks like it's glowing.
   - **Suggested fix (my idea, not in your code yet):** multiply the water term by a light factor:
     $$
     \text{lit}=\text{ambientIntensity}+0.15\,\text{sunIntensity}\max(l_y,0)
     $$
     $$
     \text{hdr}=\mathrm{mix}(\text{hdr},\ (\text{water}+(.25,.36,.43)F_w)\cdot\text{lit},\ 0.80)
     $$

2. **Snow never shows up.**
   - The terrain is made as \(2.4\times\) noise, and the noise is clamped to between −1 and 1, so the highest a hill can be is 2.4.
   - But the snow formula `smoothstep(3.2, 5.4, y)` only starts at a height of 3.2, so it's always 0.
   - This holds unless you sculpt hills higher than 3.2 by hand.

3. **Waves don't change the lighting.**
   - The vertex wave (5.1) moves the water surface, but the normal stays \((0,1,0)\), so the lighting and Fresnel can't see the waves.
   - The vertex wave also moves with \(+1.4t\) while the color ripple moves with \(-0.8t\), so the two travel in opposite directions.

4. **The buoyancy area is fixed.**
   - Section 6 uses `waterCenter_` and `waterSurfaceY_` from startup (`MEC:282-283,1571-1575`).
   - If you move the water actor in the editor, things still float in the old spot.
   - The bug-fix patch fixes this, but this source snapshot still has the old code.

5. **Mouse look might be backwards (check by playing).**
   - Look at `MEC:1329` and `1606-1607` together. Moving the mouse right makes \(\psi\) smaller, which turns `forward` toward −X, but `right` is +X at \(\psi=0\).
   - If turning feels flipped, change `-=` to `+=`.

6. **Splash "randomness" repeats.**
   - The seed is the particle count (`MEC:711-712`), so every splash of the same size looks exactly the same.
