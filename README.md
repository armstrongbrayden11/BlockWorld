# mini_minecraft.py
# A simple Minecraft-like voxel game using Ursina
# Requires: pip install ursina

from ursina import *
from ursina.prefabs.first_person_controller import FirstPersonController
import random
import math

app = Ursina()
window.title = 'MiniCraft (Ursina)'
window.borderless = False
window.fullscreen = False
window.exit_button.visible = False
window.fps_counter.enabled = True

# block textures/colors (no external images)
BLOCK_COLORS = {
    'grass': color.rgb(106, 176, 74),
    'dirt' : color.rgb(134, 96, 67),
    'stone': color.rgb(150, 150, 150),
    'sand' : color.rgb(237, 225, 181),
    'wood' : color.rgb(156, 102, 31),
    'leaves': color.rgb(34, 139, 34),
}

BLOCK_TYPES = list(BLOCK_COLORS.keys())

# Voxel entity
class Voxel(Button):
    def __init__(self, position=(0,0,0), block_type='grass'):
        super().__init__(
            parent=scene,
            position=position,
            model='cube',
            origin_y=0.5,
            texture=None,          # no image texture, use color
            color=BLOCK_COLORS.get(block_type, color.white),
            scale=1
        )
        self.block_type = block_type

    def input(self, key):
        # allow interaction via mouse handled in update() instead
        pass

# Basic chunk / world generator (small for performance)
CHUNK_SIZE = 24
WORLD_HEIGHT = 12

voxels = {}  # dict of (x,y,z) -> Voxel

def set_block(x, y, z, block_type='grass'):
    if (x,y,z) in voxels:
        return
    v = Voxel(position=(x,y,z), block_type=block_type)
    voxels[(x,y,z)] = v

def remove_block(x, y, z):
    v = voxels.pop((x,y,z), None)
    if v:
        destroy(v)

def generate_terrain(seed=None):
    if seed is None:
        seed = random.randint(0, 99999)
    random.seed(seed)
    # simple procedural terrain using combined sine waves + randomness
    freq1 = 0.12
    freq2 = 0.07
    amp1 = 3.0
    amp2 = 1.5
    for x in range(-CHUNK_SIZE//2, CHUNK_SIZE//2):
        for z in range(-CHUNK_SIZE//2, CHUNK_SIZE//2):
            # height map
            h = int((math.sin(x*freq1) * amp1) + (math.cos(z*freq2) * amp2) + (CHUNK_SIZE//5) + random.uniform(-1,1))
            h = max(1, min(WORLD_HEIGHT-1, h))
            for y in range(0, h):
                # top block grass, below dirt, some stone deeper
                if y >= h-1:
                    btype = 'grass'
                elif y >= h-4:
                    btype = 'dirt'
                else:
                    btype = 'stone'
                set_block(x, y, z, btype)
            # add some sand near lower areas
            if h <= 3 and random.random() < 0.2:
                set_block(x, h, z, 'sand')
            # some trees
            if random.random() < 0.03 and h > 3:
                make_tree(x, h, z)

def make_tree(x, y, z):
    trunk_height = random.randint(3,5)
    for i in range(trunk_height):
        set_block(x, y+i, z, 'wood')
    # leaves cube
    for lx in range(-2,3):
        for ly in range(0,3):
            for lz in range(-2,3):
                if abs(lx) + abs(ly) + abs(lz) < 5:
                    set_block(x+lx, y+trunk_height-1+ly, z+lz, 'leaves')

# generate initial world
generate_terrain(seed=101)

# Player (first person)
player = FirstPersonController()
player.gravity = 0.5
player.cursor.visible = True
player.position = (0, WORLD_HEIGHT+2, 0)
player.speed = 6

# Simple hotbar and block selection UI
hotbar_index = 0

hotbar = []
for i, bname in enumerate(BLOCK_TYPES):
    txt = Text(bname[0].upper(), parent=camera.ui, x=-0.85 + i*0.18, y=-0.45, background=True, scale=2)
    hotbar.append(txt)

def highlight_hotbar():
    for i, t in enumerate(hotbar):
        if i == hotbar_index:
            t.background.color = color.yellow.tint(-.2)
        else:
            t.background.color = color.gray

highlight_hotbar()

# Raycast based block interaction
hold_to_place = False
place_cooldown = 0.08
last_place_time = time.time()

def update():
    global hotbar_index, last_place_time

    # scroll hotbar
    if held_keys['scroll up'] or mouse.scroll.y > 0:
        hotbar_index = (hotbar_index - 1) % len(BLOCK_TYPES)
        highlight_hotbar()
        mouse.scroll = Vec2(0,0)
    if held_keys['scroll down'] or mouse.scroll.y < 0:
        hotbar_index = (hotbar_index + 1) % len(BLOCK_TYPES)
        highlight_hotbar()
        mouse.scroll = Vec2(0,0)

    # block highlight and interaction
    if mouse.hovered_entity:
        pass  # don't rely solely on hovered_entity; use raycast below for accuracy

    # raycast from camera forward
    origin = camera.world_position
    direction = camera.forward
    hit_info = raycast(origin, direction, distance=8, traverse_target=scene, ignore=(player,))

    # show a thin translucent cube at targeted face or at placement position
    for e in scene.entities:
        if isinstance(e, Entity) and e.name == 'target_marker':
            destroy(e)
    if hit_info.hit:
        # highlight block being looked at
        block_pos = tuple([int(round(c)) for c in hit_info.entity.position])
        marker = Entity(model='cube', color=color.rgba(255,255,255,60), position=block_pos, scale=1.01, name='target_marker')
        # place position = adjacent to face normal
        place_pos = (block_pos[0] + int(hit_info.normal.x),
                     block_pos[1] + int(hit_info.normal.y),
                     block_pos[2] + int(hit_info.normal.z))

        # left click to remove
        if mouse.left and time.time() - last_place_time > place_cooldown:
            remove_block(*block_pos)
            last_place_time = time.time()

        # right click to place
        if mouse.right and time.time() - last_place_time > place_cooldown:
            chosen = BLOCK_TYPES[hotbar_index]
            # don't allow placing inside player bounding box (simple check)
            px, py, pz = player.position
            if not (place_pos[0] == int(px) and place_pos[1] == int(py) and place_pos[2] == int(pz)):
                set_block(*place_pos, block_type=chosen)
            last_place_time = time.time()

    # keyboard hotkeys for quick selection
    for i in range(min(9, len(BLOCK_TYPES))):
        if held_keys[str(i+1)]:
            hotbar_index = i
            highlight_hotbar()

# simple world-saving (press 'k' to save a tiny snapshot)
def input(key):
    if key == 'k':
        save_world()

def save_world(filename='world_snapshot.txt'):
    with open(filename, 'w') as f:
        for (x,y,z), v in voxels.items():
            f.write(f"{x},{y},{z},{v.block_type}\n")
    print(f"Saved {len(voxels)} blocks to {filename}")

# Simple on-screen instructions
instructions = Text(
    "WASD: move  Space: jump  Mouse: look\nLeft click: remove block  Right click: place block\n1-9: select block  K: save snapshot",
    parent=camera.ui, x=-0.7, y=0.45, scale=1.2
)

app.run()
