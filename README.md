# Mesh-cut 

An interactive cloth simulation with real-time cutting and adaptive mesh refinement, built with vanilla JavaScript and HTML5 Canvas. Experience realistic cloth physics with the ability to cut and tear the fabric in real-time.

![Cloth Simulation](https://img.shields.io/badge/Physics-Verlet%20Integration-blue) ![Canvas](https://img.shields.io/badge/Rendering-HTML5%20Canvas-green)

## 🎮 Features

- **Real-time Cloth Physics**: Verlet integration-based particle system
- **Interactive Cutting**: Click and drag to cut through the cloth
- **Adaptive Mesh Refinement**: Automatic mesh subdivision on hover for precise cutting
- **Tearing Simulation**: Realistic cloth tearing when edges are overstretched
- **Visual Feedback**: Dynamic shading based on surface normals
- **Performance Optimized**: Efficient constraint solving with spatial optimizations

## 🚀 Quick Start

### Prerequisites

No dependencies required! This is a pure vanilla JavaScript implementation that runs directly in any modern web browser.

### Running the Project

1. **Clone or download** this repository
2. **Open `index.html`** in any modern web browser (Chrome, Firefox, Edge, Safari)
   - Simply double-click the file, or
   - Right-click → Open with → Your preferred browser
3. **Interact with the cloth**:
   - **Hover** over the cloth to refine the mesh for precise cutting
   - **Click and drag** to cut through the cloth
   - Watch the cloth fall and react to gravity

### Alternative: Local Server (Optional)

For better development experience, you can use a local server:

```bash
# Using Python 3
python -m http.server 8000

# Using Python 2
python -m SimpleHTTPServer 8000

# Using Node.js (if you have http-server installed)
npx http-server

# Using PHP
php -S localhost:8000
```

Then navigate to `http://localhost:8000` in your browser.

## 📐 Physics & Mathematics

### Core Physics Engine

The simulation uses a **Verlet Integration** particle system with **constraint-based dynamics** to simulate realistic cloth behavior.

#### 1. Verlet Integration

Verlet integration is a numerical method for solving Newton's equations of motion. It's particularly well-suited for constraint-based systems because it maintains energy stability.

**Position Update:**
```
velocity = (current_position - old_position) * damping
old_position = current_position
current_position += velocity + acceleration * dt²
```

In code (lines 384-387):
```javascript
let vx = (p.x - p.oldx) * 0.99;  // Velocity with damping
let vy = (p.y - p.oldy) * 0.99;
p.oldx = p.x; p.oldy = p.y;       // Store old position
p.x += vx; 
p.y += vy + CONFIG.gravity * 0.0005;  // Apply gravity
```

**Why Verlet?**
- Energy-conserving (stable over long simulations)
- No explicit velocity storage needed
- Natural for constraint-based systems
- Time-reversible

#### 2. Constraint-Based Dynamics

The cloth is modeled as a network of **distance constraints** between particles. Each constraint maintains a fixed rest length between two points.

**Constraint Solving (lines 91-106):**

The constraint solver uses a **relaxation method** (iterative constraint satisfaction):

1. **Calculate current distance:**
   ```javascript
   const dx = p1.x - p2.x;
   const dy = p1.y - p2.y;
   const dist = Math.sqrt(dx * dx + dy * dy);
   ```

2. **Compute correction factor:**
   ```javascript
   const diff = (dist - restLength) / dist;
   const correction = clamp(diff * 0.5, -0.5, 0.5);
   ```

3. **Apply correction to both points:**
   ```javascript
   if (!p1.pinned) {
     p1.x -= dx * correction;
     p1.y -= dy * correction;
   }
   if (!p2.pinned) {
     p2.x += dx * correction;
     p2.y += dy * correction;
   }
   ```

**Multiple Iterations:** The solver runs `CONFIG.physicsIterations` (16) times per frame to ensure constraints converge, creating stable, realistic cloth behavior.

#### 3. Gravity

Gravity is applied as a constant downward acceleration:
```javascript
p.y += vy + CONFIG.gravity * 0.0005;
```

Where `CONFIG.gravity = 1000` pixels/second², scaled by the timestep.

#### 4. Damping

Velocity damping (air resistance) prevents infinite oscillation:
```javascript
let vx = (p.x - p.oldx) * 0.99;  // 1% damping per frame
```

The `0.99` factor means 1% of velocity is lost each frame, simulating air resistance.

### Mesh Structure

#### Initial Grid Generation (lines 162-191)

The cloth starts as a regular grid:
- **24 columns × 16 rows** of particles
- **30-pixel spacing** between particles
- **Top row pinned** (fixed in place) to simulate hanging cloth
- **Triangular mesh** created by connecting particles in a checkerboard pattern

Each square in the grid is divided into two triangles, creating a stable mesh structure.

#### Constraint Network

Each triangle edge becomes a constraint:
- **Horizontal edges**: Connect adjacent particles in the same row
- **Vertical edges**: Connect particles in adjacent rows
- **Diagonal edges**: Connect diagonal neighbors (stabilizes the mesh)

### Adaptive Mesh Refinement

When you hover near the cloth, the mesh automatically subdivides to allow for more precise cutting.

#### Refinement Algorithm (lines 225-297)

1. **Triangle Selection:**
   - Find triangles whose centroids are within `refineRadius` of the mouse path
   - Uses point-to-segment distance calculation (lines 327-333)

2. **Edge Splitting:**
   - Each edge of selected triangles is split at its midpoint
   - New midpoint is added as a particle
   - Original constraint is deactivated, two new constraints are created

3. **Triangle Subdivision:**
   - Each triangle is split into 4 smaller triangles
   - Maintains mesh connectivity
   - Prevents infinite refinement with generation limit (max 3 levels)

4. **Midpoint Caching:**
   - Uses `midpointMap` to prevent duplicate midpoints when edges are shared
   - Ensures mesh consistency

**Mathematical Details:**

**Point-to-Segment Distance (lines 327-333):**
```javascript
// Project point p onto line segment v-w
const l2 = distance(v, w)²
const t = clamp(dot(p - v, w - v) / l2, 0, 1)
const projection = v + t * (w - v)
return distance(p, projection)
```

This ensures refinement happens along the entire mouse path, not just at discrete points.

### Cutting Mechanism

#### Constraint Deactivation (lines 299-307)

When you click, constraints within the cut radius are deactivated:
```javascript
const distSq = (midpoint.x - clickX)² + (midpoint.y - clickY)²;
if (distSq < cutRadius²) {
  constraint.active = false;
}
```

#### Triangle Removal (lines 314-324)

Triangles whose centroids are within the cut radius are also deactivated, preventing "dust" (tiny disconnected fragments).

#### Interpolation Fix (lines 344-374)

To prevent gaps when cutting quickly, the mouse path is interpolated:
```javascript
const steps = Math.ceil(distance / 10);  // One step per 10 pixels
for (let i = 1; i <= steps; i++) {
  const t = i / steps;
  const x = lerp(oldX, newX, t);
  const y = lerp(oldY, newY, t);
  cutAt(x, y);
}
```

This ensures continuous cutting even with fast mouse movements.

### Tearing Simulation

When constraints are overstretched, triangles are automatically hidden to simulate tearing (lines 398-402):

```javascript
const d1 = distance(p1, p2);
const d2 = distance(p2, p3);
const d3 = distance(p3, p1);

if (d1 > tearThreshold || d2 > tearThreshold || d3 > tearThreshold) {
  continue;  // Skip rendering this triangle
}
```

When any edge exceeds `CONFIG.tearThreshold` (70 pixels), the triangle is not rendered, creating a visual tear.

### Rendering & Shading

#### Normal Calculation (lines 404-408)

Surface normals are calculated using the **cross product** of triangle edges:

```javascript
const ax = p2.x - p1.x, ay = p2.y - p1.y;  // Edge 1
const bx = p3.x - p1.x, by = p3.y - p1.y;  // Edge 2
const z = ax * by - ay * bx;  // 2D cross product (gives signed area)
```

The cross product magnitude represents the triangle's area and orientation.

#### Shading (lines 408-411)

Shading is based on the triangle's orientation:
```javascript
const shade = 0.5 + (Math.abs(z) / 1500);
const c = colorVal * shade;
fillStyle = `rgb(${c * 0.2}, ${c * 0.8}, ${c})`;
```

- Larger `|z|` (more perpendicular to view) → brighter
- Smaller `|z|` (more edge-on) → darker
- Creates depth perception and 3D-like appearance

## ⚙️ Configuration

All physics parameters are in the `CONFIG` object (lines 55-63):

```javascript
const CONFIG = {
  gravity: 1000,              // Gravity acceleration (pixels/s²)
  physicsIterations: 16,      // Constraint solver iterations per frame
  refineRadius: 40,           // Mesh refinement radius (pixels)
  cutRadius: 20,              // Cutting radius (pixels)
  minEdgeLength: 8,           // Minimum edge length before splitting
  tearThreshold: 70,          // Maximum edge length before tearing
  spacing: 30                 // Initial particle spacing (pixels)
};
```

### Tuning Tips

- **Increase `gravity`**: Faster falling, more dramatic motion
- **Increase `physicsIterations`**: More stable, but slower (try 8-32)
- **Increase `refineRadius`**: Larger refinement area on hover
- **Increase `cutRadius`**: Easier cutting, less precision
- **Decrease `tearThreshold`**: Cloth tears more easily
- **Decrease `spacing`**: Denser initial mesh, more particles

## 🏗️ Architecture

### Class Structure

- **`Point`**: Represents a particle in the cloth
  - Position (`x`, `y`)
  - Previous position (`oldx`, `oldy`) for Verlet integration
  - `pinned` flag for fixed points

- **`Constraint`**: Maintains distance between two points
  - Rest length
  - Active state (deactivated when cut)
  - Midpoint reference (for refinement)

- **`Triangle`**: Rendered triangle face
  - Three point references
  - Active state
  - Generation level (for refinement limit)
  - Color value

- **`ClothSim`**: Main simulation class
  - Manages all points, constraints, and triangles
  - Handles input events
  - Runs physics loop

### Data Structures

- **`constraintMap`**: Maps edge keys to constraints for O(1) lookup
- **`midpointMap`**: Prevents duplicate midpoints during refinement
- **Edge keys**: Format `"id1_id2"` where `id1 < id2` ensures uniqueness

## 🎯 Performance Optimizations

1. **Spatial Culling**: Only processes triangles/constraints near the mouse
2. **Constraint Map**: O(1) constraint lookup instead of O(n) search
3. **Midpoint Caching**: Prevents redundant edge splits
4. **Active Flags**: Inactive triangles/constraints are skipped
5. **Generation Limit**: Prevents infinite mesh refinement

## 🔬 Mathematical Concepts Used

1. **Verlet Integration**: Numerical ODE solver
2. **Constraint Relaxation**: Iterative constraint satisfaction
3. **Linear Interpolation**: Smooth mouse path interpolation
4. **Point-to-Segment Distance**: Geometric distance calculation
5. **Cross Product**: Normal calculation and shading
6. **Euclidean Distance**: Spatial queries and constraint solving
7. **Clamping**: Numerical stability

## 📝 Code Highlights

### Key Algorithms

- **Constraint Solver**: Iterative relaxation (lines 391-393)
- **Mesh Refinement**: Adaptive subdivision (lines 225-297)
- **Cutting**: Spatial query with radius (lines 299-325)
- **Rendering**: Cross-product based shading (lines 395-417)

## 🐛 Known Limitations

- 2D simulation only (no 3D depth)
- No collision detection with other objects
- No wind or external forces (except gravity)
- Fixed top row (cannot be cut)
- Mesh refinement limited to 3 generations

## 🚧 Future Enhancements

- 3D cloth simulation
- Wind and external forces
- Collision detection
- Multiple cloth pieces
- Save/load cloth states
- Different cloth materials (stiffness, damping)

## 📄 License

This project is open source and available for educational purposes.

## 👨‍💻 Author

Created as part of AR/VR coursework at PES University.

---

**Enjoy cutting! ✂️**

