Rotavator parametric CAD builder (CadQuery script)

How to use:
1) Install CadQuery (recommended: CadQuery 2.x) or use FreeCAD with CadQuery workbench.
   - pip install cadquery
   - or follow CadQuery installation docs: https://cadquery.readthedocs.io
2) Save this file as rotavator_cadquery.py and run with cq-editor or a Python environment where cadquery is available.
   Example (cq-editor): open the file and run; or from script you can run python -m cadquery rotavator_cadquery.py
3) The script builds parametric parts (frame, gearbox placeholder, rotor shaft + flanges, blades, side covers) and exports STEP files to the working directory (./output).

Notes:
- This script creates simplified but industrially-structured geometry suitable for downstream detailing in SolidWorks/FreeCAD.
- Bolts are represented as simplified cylinders. Bearings and internal gearbox gears are left as placeholders but housings are created.
- Modify parameters in the PARAMETERS block to match your exact machine.

"""

# PARAMETERS - change these to match your machine
WIDTH = 1800.0         # working width in mm (e.g., 1800 mm = 1.8 m ~ 6 ft)
ROTOR_DIAMETER = 300.0 # rotor drum diameter in mm
ROTOR_LENGTH = WIDTH   # rotor length equals working width
NUM_SECTIONS = 6       # number of flange sections along the rotor
BLADES_PER_SECTION = 6 # blades per flange (helical stagger will be simulated by offset)
BLADE_LENGTH = 160.0   # length of J-type blade
BLADE_WIDTH = 40.0     # blade width
BLADE_THICK = 8.0      # blade thickness
FRAME_HEIGHT = 420.0   # overall frame height
FRAME_THICK = 10.0
PTO_DIAMETER = 35.0
OUTPUT_FOLDER = './output'

# Import CadQuery
import cadquery as cq
import os

os.makedirs(OUTPUT_FOLDER, exist_ok=True)

# Helper: export
def export_shape(shape, name):
    filepath = os.path.join(OUTPUT_FOLDER, f"{name}.step")
    cq.exporters.export(shape, filepath)
    print(f"Exported: {filepath}")

# 1) Create main frame (rectangular box with cutouts)
frame = (
    cq.Workplane("XY")
    .box(WIDTH + 200, FRAME_THICK, FRAME_HEIGHT)  # wide base beam
    .translate((0, 0, FRAME_HEIGHT/2))
)
# add top A-bracket mount
a_bracket = (
    cq.Workplane("XY")
    .workplane(offset=FRAME_HEIGHT/2)
    .transformed(offset=(0,0,0))
    .rect(200, 140)
    .extrude(40)
)
frame = frame.union(a_bracket)

# cutouts for rotor to pass under frame
frame = frame.faces(
    ">Z"
).workplane(offset=-30).rect(WIDTH, FRAME_THICK*2).cutThruAll()

export_shape(frame, 'frame')

# 2) Rotor drum and flanges
# rotor shaft
shaft_diameter = 60.0
shaft = cq.Workplane("XY").cylinder(shaft_diameter, ROTOR_LENGTH + 120).rotate((0,0,0),(0,1,0),90)

# rotor drum (cylindrical shell)
drum = (
    cq.Workplane("XZ")
    .circle(ROTOR_DIAMETER/2)
    .extrude(ROTOR_LENGTH + 120)
    .faces("<Z").chamfer(3)
)

# flanges along rotor
flange_thick = 16.0
flange = (
    cq.Workplane("XZ")
    .circle(ROTOR_DIAMETER/2 + 20)
    .extrude(flange_thick)
)

assembly = drum.union(shaft)
for i in range(NUM_SECTIONS):
    x_pos = -ROTOR_LENGTH/2 + (i + 0.5) * (ROTOR_LENGTH / NUM_SECTIONS)
    f = flange.translate((0, x_pos, 0))
    assembly = assembly.union(f)

export_shape(assembly, 'rotor_assembly')

# 3) Blade geometry (J-type blade) - parametric
import math

def make_j_blade():
    # Straight rectangular blade with a curled end to emulate J-shape
    base = (
        cq.Workplane("XY")
        .rect(BLADE_LENGTH, BLADE_WIDTH)
        .extrude(BLADE_THICK)
    )
    # curled tip
    tip = (
        cq.Workplane("XY")
        .workplane(offset=BLADE_THICK)
        .move(BLADE_LENGTH/2 - 30, 0)
        .threePointArc((BLADE_LENGTH/2, 30), (BLADE_LENGTH/2 + 30, 0))
        .extrude(BLADE_THICK)
    )
    blade = base.union(tip)
    # thin the root area for weld face
    blade = blade.translate((0,0,0))
    return blade

blade_solid = make_j_blade()
export_shape(blade_solid, 'single_j_blade')

# 4) Create blade flanges (tabs) and place blades on rotor
blade_hole = 14.0
blade_mounts = cq.Workplane("XZ")

blade_instances = []
for sect in range(NUM_SECTIONS):
    x_pos = -ROTOR_LENGTH/2 + (sect + 0.5) * (ROTOR_LENGTH / NUM_SECTIONS)
    # helical staggering angle per section
    for b in range(BLADES_PER_SECTION):
        angle = (360.0 / BLADES_PER_SECTION) * b + sect*15
        # position blade at drum radius
        r = ROTOR_DIAMETER/2 + BLADE_LENGTH/6
        # compute coordinates
        rad = math.radians(angle)
        y = x_pos
        z = r * math.cos(rad)
        x = r * math.sin(rad)
        # orient blade normal to drum (approx)
        bl = blade_solid.rotate((0,0,0),(0,1,0),-90)
        bl = bl.rotate((0,0,0),(1,0,0), angle)
        bl = bl.translate((x, y, z))
        blade_instances.append(bl)

# merge a few blades into a subassembly
blade_group = blade_instances[0]
for inst in blade_instances[1:]:
    blade_group = blade_group.union(inst)

export_shape(blade_group, 'blade_group')

# 5) Side covers and gearbox housing (simplified)
# left and right side covers
side_cover = (
    cq.Workplane("YZ")
    .box(ROTOR_DIAMETER + 160, 80, FRAME_HEIGHT)
    .translate((ROTOR_LENGTH/2 + 60, 0, FRAME_HEIGHT/2))
)
side_cover_left = side_cover.mirror("YZ")
side_cover_right = side_cover

export_shape(side_cover_left, 'side_cover_left')
export_shape(side_cover_right, 'side_cover_right')

# gearbox placeholder
gearbox = (
    cq.Workplane("XY")
    .box(220, 140, 140)
    .faces(
        ">Z"
    ).workplane(offset=-20).hole(shaft_diameter + 4)
    .translate((0, ROTOR_LENGTH/2 + 60, 70))
)

export_shape(gearbox, 'gearbox_housing')

# 6) PTO stub and coupling
pto = cq.Workplane("XY").cylinder(PTO_DIAMETER, 120).translate((0, ROTOR_LENGTH/2 + 120, FRAME_HEIGHT/2 - 20))
export_shape(pto, 'pto_stub')

# 7) Simple bolts array (representative)
bolt = cq.Workplane("XY").cylinder(10, 30)
bolts = cq.Workplane("XY")
for i in range(6):
    x = -250 + i*100
    bolts = bolts.union(bolt.translate((x, ROTOR_LENGTH/2 + 20, FRAME_HEIGHT - 30)))
export_shape(bolts, 'bolts')

# 8) Assembly (frame + rotor + blades + side covers + gearbox)
full_assembly = frame.union(assembly).union(blade_group).union(side_cover_left).union(side_cover_right).union(gearbox).union(pto)
export_shape(full_assembly, 'rotavator_full_assembly')

print('All STEP files exported to', OUTPUT_FOLDER)
