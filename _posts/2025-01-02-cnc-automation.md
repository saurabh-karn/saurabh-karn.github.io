---
title: "Automating CNC Machining: From CAD to Machine Code"
description: "Building an algorithm to automatically generate G-Code from CAD models for CNC lathes"
tags: [manufacturing, automation, algorithms]
---

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>

<style>
#threejs-container {
    width: 100%;
    height: 400px;
    border-radius: 8px;
    overflow: hidden;
    margin: 20px 0;
    background: #1a202c;
}

.help-text {
    text-align: center;
    color: #6b7280;
    font-size: 0.875rem;
    margin-bottom: 20px;
}

/* Code block enhancements */
.code-wrapper {
    position: relative;
    margin: 24px 0;
}

.code-tag {
    display: inline-block;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    padding: 6px 16px;
    border-radius: 6px 6px 0 0;
    font-size: 0.875rem;
    font-weight: 600;
    letter-spacing: 0.5px;
    margin-bottom: -1px;
}

.code-tag.stl {
    background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
}

.code-tag.gcode {
    background: linear-gradient(135deg, #4facfe 0%, #00f2fe 100%);
}

.code-tag.pseudocode {
    background: linear-gradient(135deg, #43e97b 0%, #38f9d7 100%);
}

.code-tag.example {
    background: linear-gradient(135deg, #fa709a 0%, #fee140 100%);
}

.code-wrapper pre {
    margin-top: 0;
    border: 2px solid #e5e7eb;
    border-radius: 0 8px 8px 8px;
    background: #1f2937 !important;
    padding: 20px !important;
    overflow-x: auto;
}

.code-wrapper pre code {
    color: #10b981;
    font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
    font-size: 0.9rem;
    line-height: 1.6;
}

/* Inline code enhancement */
code {
    background: #f3f4f6;
    color: #be185d;
    padding: 2px 6px;
    border-radius: 3px;
    font-size: 0.9em;
    font-weight: 600;
}

pre code {
    background: transparent !important;
    color: #10b981 !important;
    padding: 0 !important;
    font-weight: normal !important;
}
</style>

## Overview

Back in my undergrad days, I set out to solve a classic manufacturing headache: how to get from a shiny CAD model to real CNC machines. You see the CNC machines understand coordinates only. You see G-Code is the standard language used to control CNC machines. It consists of a series of commands, each describing a sepcific movement or action for the machine. These commands tell the CNC machine:
1. Where to move
2. How fast to move
3. What operations to perform

**Common G-Code Commands:**
- `G0` or `G00`: Rapid positioning (move quickly to a position)
- `G1` or `G01`: Linear interpolation (move in a straight line)
- `G2` / `G3`: Circular interpolation (move in a circle or arc)
- `M3` / `M05`: Spindle on / Spindle off

An example of G Code would be:

<div class="code-wrapper">
<span class="code-tag example">G-Code Example</span>
<pre><code>G21         ; Set units to millimeters
G90         ; Absolute positioning
M3 S1200    ; Start spindle at 1200 rpm

G0 X30 Z2   ; Rapidly move to starting position (diameter 30mm, 2mm ahead of material)
G1 Z0 F0.25 ; Feed to beginning of material at 0.25 mm/rev
G1 X20      ; Turn down to diameter 20mm

G1 Z-50     ; Feed along the Z axis to the end of the cut
G0 X100     ; Rapid move tool away from part

M5          ; Stop spindle
M30         ; Program end</code></pre>
</div>

## The Problem

In a typical manufacturing workflow, there's a critical disconnect i.e. engineers design parts in CAD software like Pro-E (now Creo), focusing on functionality, dimensions, and tolerances. A machinist would then receives the design and must manually write G-Code (machine instructions) to actually make the part. What if we could translate the model directly into G-Codes?

---

## The Part: Stepped Tapered Pyramid

<div id="threejs-container"></div>
<p class="help-text">Drag to rotate • This part has alternating cylindrical and tapered sections</p>

This geometry is common in industrial applications: transmission shafts, hydraulic pistons, valve stems, and aerospace components. The challenge is machining precise transitions between different diameters while maintaining tight tolerances.

---

## Step 1: The STL File

When you export a CAD model from Pro-E, you get an **STL file** — a format that represents the 3D surface as a mesh of triangular facets. Each triangle is defined by three vertices and a normal vector.

<div class="code-wrapper">
<span class="code-tag stl">STL File Format</span>
<pre><code>solid stepped_tapered_pyramid
  facet normal 0.0 0.0 1.0
    outer loop
      vertex 10.0 0.0 0.0
      vertex 8.66 5.0 0.0
      vertex 8.66 -5.0 0.0
    endloop
  endfacet
  
  facet normal 0.866 0.5 0.0
    outer loop
      vertex 10.0 0.0 0.0
      vertex 8.66 5.0 0.0
      vertex 8.66 5.0 40.0
    endloop
  endfacet
  
  ... (thousands more triangles)
  
endsolid stepped_tapered_pyramid</code></pre>
</div>

For our stepped pyramid, the STL file contains **thousands of triangles** describing every surface. This is just geometric data — not machine instructions.

---

## Step 2: The Conversion Algorithm

Here's where the magic happens. My algorithm reads the STL file and automatically generates CNC machine instructions:

### 1. Parse STL Triangles
Read all triangle vertices and identify the geometric patterns. For cylindrical shapes, triangles cluster around circular cross-sections.

### 2. Identify Segments
Detect cylindrical sections (constant radius) vs. tapered sections (changing radius). Calculate start/end positions and radii for each segment.

### 3. Calculate Toolpaths
For each segment, generate circular toolpaths at the appropriate radius. Use circular interpolation (G02/G03 commands) for efficient machining.

### 4. Generate G-Code
Convert toolpaths into machine commands with proper feed rates, spindle speeds, and safe approach/retract heights.

**Key Algorithm Features:**
- Handles limited-axis CNC machines (2.5D machining)
- Optimizes cutting paths to minimize tool travel
- Accounts for tool diameter and safe clearances
- Generates helical interpolation for tapered transitions

### Algorithm Pseudocode

Here's the core logic in pseudocode:

<div class="code-wrapper">
<span class="code-tag pseudocode">Algorithm Pseudocode</span>
<pre><code>ALGORITHM: STL to G-Code Converter

INPUT: STL file containing triangular mesh
OUTPUT: G-Code file with machine instructions

FUNCTION ParseSTL(stl_file):
    vertices = empty list
    FOR each line in stl_file:
        IF line contains "vertex":
            EXTRACT x, y, z coordinates
            ADD (x, y, z) to vertices
    RETURN vertices

FUNCTION IdentifySegments(vertices):
    // Group vertices by Z-coordinate to find cross-sections
    z_groups = empty dictionary
    
    FOR each vertex in vertices:
        z = ROUND(vertex.z, tolerance=0.1mm)
        ADD vertex to z_groups[z]
    
    // Calculate radius at each Z-level
    segments = empty list
    FOR each z in SORTED(z_groups.keys()):
        points = z_groups[z]
        radii = empty list
        FOR each point in points:
            radius = SQRT(point.x² + point.y²)
            ADD radius to radii
        avg_radius = MEAN(radii)
        ADD (z, avg_radius) to segments
    
    // Merge consecutive segments with same radius
    merged = empty list
    FOR i = 0 to LENGTH(segments):
        IF i == 0 OR |segments[i].radius - segments[i-1].radius| > 0.5mm:
            CREATE new segment:
                z_start = segments[i].z
                z_end = segments[i].z
                radius_start = segments[i].radius
                radius_end = segments[i].radius
            ADD segment to merged
        ELSE:
            UPDATE merged[LAST].z_end = segments[i].z
            UPDATE merged[LAST].radius_end = segments[i].radius
    
    RETURN merged

FUNCTION GenerateGCode(segments, output_file):
    WRITE header:
        "G21"        // Set units to millimeters
        "G90"        // Absolute positioning
        "S1200 M03"  // Spindle on at 1200 RPM
        "F150"       // Feed rate 150 mm/min
    
    FOR each segment in segments:
        radius = segment.radius_start
        z_start = segment.z_start
        z_end = segment.z_end
        
        // Position tool at starting point
        WRITE "G00 X{radius} Y0.000"     // Rapid move to radius
        WRITE "G01 Z{z_start}"            // Move to Z start position
        
        // Check if cylindrical or tapered
        IF |segment.radius_start - segment.radius_end| < 0.1mm:
            // Cylindrical section - complete circle at constant radius
            WRITE "G02 X{radius} Y0.000 I{-radius} J0.000"
        ELSE:
            // Tapered section - helical interpolation
            radius_end = segment.radius_end
            WRITE "G02 X{radius_end} Y0.000 Z{z_end} I{-radius} J0.000"
        
        WRITE "G00 Z20.0"  // Retract to safe height
    
    WRITE footer:
        "M05"       // Spindle off
        "G00 Z50.0" // Final safe height
        "M30"       // End program

MAIN:
    vertices = ParseSTL("stepped_pyramid.stl")
    segments = IdentifySegments(vertices)
    GenerateGCode(segments, "output.nc")</code></pre>
</div>

**Key Steps:**
1. **Parse** - Extract all (x, y, z) coordinates from STL triangles
2. **Group** - Organize vertices by Z-height to identify cross-sections
3. **Analyze** - Calculate radius at each level (√(x² + y²))
4. **Merge** - Combine consecutive layers with similar radii into segments
5. **Generate** - Output G-Code with circular interpolation commands

The real implementation I used in my undergraduate project was more sophisticated, handling tool compensation, collision detection, and optimized toolpath planning, but this captures the essential concept.

---

## Step 3: The G-Code Output

The final output is **G-Code** — the language CNC machines speak. Instead of spending hours writing this by hand, the algorithm generates it automatically:

<div class="code-wrapper">
<span class="code-tag gcode">Generated G-Code Output</span>
<pre><code>; Stepped Tapered Pyramid - Auto-Generated G-Code
; Tool: 6mm End Mill | Material: Aluminum

G21                      ; Set units to millimeters
G90                      ; Absolute positioning
G17                      ; XY plane selection
F150                     ; Feed rate 150 mm/min

; Machine initialization
S1200 M03               ; Spindle on, 1200 RPM
G00 Z20.0               ; Move to safe height

; ===== SEGMENT 1: Ø20mm Cylinder (0-40mm) =====
G00 X10.000 Y0.000      ; Position at radius
G01 Z5.0                ; Approach height
G01 Z0.0                ; Cut to depth
G02 X10.000 Y0.000 I-10.000 J0.000  ; Complete circle
G00 Z20.0               ; Retract to safe height

; ===== SEGMENT 2: Taper Ø20→Ø40mm (40-50mm) =====
G00 X10.000 Y0.000
G01 Z40.0
; Helical interpolation for taper
G02 X15.000 Y0.000 Z45.0 I-10.000 J0.000
G02 X20.000 Y0.000 Z50.0 I-15.000 J0.000
G00 Z20.0

; ===== SEGMENT 3: Ø40mm Cylinder (50-90mm) =====
G00 X20.000 Y0.000
G01 Z50.0
G02 X20.000 Y0.000 I-20.000 J0.000
G00 Z20.0

; Program end
M05                     ; Spindle off
G00 Z50.0               ; Final safe height
M30                     ; End of program</code></pre>
</div>

---

## Impact

1. **Time Savings**: Reduced programming time from 2-3 hours to 30 seconds
2. **Eliminates Human Error**: No typos, wrong coordinates, or missing safety commands
3. **Enables Rapid Iteration**: Design changes can be quickly reflected in manufacturing
4. **Democratizes Manufacturing**: Junior engineers can produce parts without years of G-Code experience
5. **Cost Reduction**: Less programming time = lower production costs
6. **Works on Legacy Equipment**: Designed for limited-axis CNC machines commonly found in workshops

---

## Real-World Applications

This type of automation is valuable across industries:

1. **Automotive**: Transmission shafts, brake rotors, piston rods
2. **Aerospace**: Landing gear components, actuator housings
3. **Medical**: Prosthetic implants, surgical instrument handles
4. **Oil & Gas**: Drill collars, valve stems, pump shafts

We did not know if this is revolutionary or just an interesting project, but it personally felt brining physical and computational world together.

---

**Technologies**: Python, STL parsing, computational geometry, CNC machining, G-Code generation

<script>
    // Three.js scene setup
    const container = document.getElementById('threejs-container');
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x1a202c);

    const camera = new THREE.PerspectiveCamera(
        75,
        container.clientWidth / container.clientHeight,
        0.1,
        1000
    );
    camera.position.set(150, 100, 150);
    camera.lookAt(0, 0, 0);

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(container.clientWidth, container.clientHeight);
    container.appendChild(renderer.domElement);

    // Lighting
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
    scene.add(ambientLight);

    const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    directionalLight.position.set(100, 100, 50);
    scene.add(directionalLight);

    const directionalLight2 = new THREE.DirectionalLight(0xffffff, 0.4);
    directionalLight2.position.set(-100, -100, -50);
    scene.add(directionalLight2);

    // Create stepped pyramid
    const segments = [
        { bottomRadius: 10, topRadius: 10, height: 0, topHeight: 40, color: 0x3b82f6 },
        { bottomRadius: 10, topRadius: 20, height: 40, topHeight: 50, color: 0x60a5fa },
        { bottomRadius: 20, topRadius: 20, height: 50, topHeight: 90, color: 0x2563eb },
        { bottomRadius: 20, topRadius: 30, height: 90, topHeight: 100, color: 0x60a5fa },
        { bottomRadius: 30, topRadius: 30, height: 100, topHeight: 140, color: 0x1d4ed8 },
        { bottomRadius: 30, topRadius: 40, height: 140, topHeight: 150, color: 0x60a5fa },
        { bottomRadius: 40, topRadius: 40, height: 150, topHeight: 200, color: 0x1e40af }
    ];

    const cylinderHeight = 200;
    segments.forEach((segment) => {
        const height = segment.topHeight - segment.height;
        const geometry = new THREE.CylinderGeometry(
            segment.topRadius,
            segment.bottomRadius,
            height,
            32
        );
        const material = new THREE.MeshPhongMaterial({
            color: segment.color,
            shininess: 30
        });
        const cylinder = new THREE.Mesh(geometry, material);
        
        cylinder.rotation.x = Math.PI / 2;
        cylinder.position.z = segment.height + height / 2 - cylinderHeight / 2;
        
        const edges = new THREE.EdgesGeometry(geometry);
        const line = new THREE.LineSegments(
            edges,
            new THREE.LineBasicMaterial({ color: 0x000000, linewidth: 1 })
        );
        cylinder.add(line);
        
        scene.add(cylinder);
    });

    // Axes
    const axesHelper = new THREE.AxesHelper(100);
    scene.add(axesHelper);

    // Axis labels
    function createTextSprite(text, color, position) {
        const canvas = document.createElement('canvas');
        const context = canvas.getContext('2d');
        canvas.width = 256;
        canvas.height = 256;
        context.font = 'Bold 100px Arial';
        context.fillStyle = color;
        context.textAlign = 'center';
        context.fillText(text, 128, 128);
        
        const texture = new THREE.CanvasTexture(canvas);
        const spriteMaterial = new THREE.SpriteMaterial({ map: texture });
        const sprite = new THREE.Sprite(spriteMaterial);
        sprite.position.copy(position);
        sprite.scale.set(20, 20, 1);
        return sprite;
    }

    scene.add(createTextSprite('X', '#ef4444', new THREE.Vector3(120, 0, 0)));
    scene.add(createTextSprite('Y', '#10b981', new THREE.Vector3(0, 120, 0)));
    scene.add(createTextSprite('Z', '#3b82f6', new THREE.Vector3(0, 0, 120)));

    // Mouse controls
    let isDragging = false;
    let previousMousePosition = { x: 0, y: 0 };

    renderer.domElement.addEventListener('mousedown', (e) => {
        isDragging = true;
        previousMousePosition = { x: e.clientX, y: e.clientY };
    });

    renderer.domElement.addEventListener('mousemove', (e) => {
        if (!isDragging) return;
        const deltaX = e.clientX - previousMousePosition.x;
        const deltaY = e.clientY - previousMousePosition.y;
        scene.rotation.y += deltaX * 0.01;
        scene.rotation.x += deltaY * 0.01;
        previousMousePosition = { x: e.clientX, y: e.clientY };
    });

    renderer.domElement.addEventListener('mouseup', () => {
        isDragging = false;
    });

    renderer.domElement.addEventListener('mouseleave', () => {
        isDragging = false;
    });

    // Animation loop
    function animate() {
        requestAnimationFrame(animate);
        renderer.render(scene, camera);
    }
    animate();

    // Handle window resize
    window.addEventListener('resize', () => {
        const width = container.clientWidth;
        const height = container.clientHeight;
        camera.aspect = width / height;
        camera.updateProjectionMatrix();
        renderer.setSize(width, height);
    });
</script>
