bl_info = {
    "name": "Vavatar_tools_KN",
    "author": "You",
    "version": (1, 0, 0),
    "blender": (4, 2, 0),
    "location": "View3D > Sidebar (N) > Shape Key / Bake",
    "description": "ARKit 52 shape-key creation & L/R split, plus projected-UV creation and auto emit-bake.",
    "category": "Object",
}

import bpy
import os
import json
import textwrap
import unicodedata
import urllib.request
from bpy.props import FloatProperty, BoolProperty, StringProperty, EnumProperty, PointerProperty
from bpy_extras.io_utils import ExportHelper


# ===========================================================================
#  Shared
# ===========================================================================
def _mesh_obj(context):
    obj = context.active_object
    if obj is None or obj.type != 'MESH':
        return None
    return obj


def _disp_width(s):
    """Display width where CJK (full/wide) chars count as 2, others as 1."""
    w = 0
    for ch in s:
        w += 2 if unicodedata.east_asian_width(ch) in ('W', 'F') else 1
    return w


def _wrap_cjk(text, max_units):
    """Wrap text by display width (CJK-aware). Breaks on spaces, hard-breaks long runs."""
    out = []
    for para in text.split('\n'):
        cur, cur_w = '', 0
        for word in para.split(' '):
            ww = _disp_width(word)
            if cur == '':
                cur, cur_w = word, ww
            elif cur_w + 1 + ww <= max_units:
                cur += ' ' + word
                cur_w += 1 + ww
            else:
                out.append(cur)
                cur, cur_w = word, ww
        if cur != '':
            out.append(cur)
    # hard-break any remaining over-long line (e.g. a long run without spaces)
    final = []
    for ln in out:
        if _disp_width(ln) <= max_units:
            final.append(ln)
            continue
        piece, pw = '', 0
        for ch in ln:
            cw = 2 if unicodedata.east_asian_width(ch) in ('W', 'F') else 1
            if piece and pw + cw > max_units:
                final.append(piece)
                piece, pw = ch, cw
            else:
                piece += ch
                pw += cw
        if piece:
            final.append(piece)
    return final or [text]


def _wrap_label(layout, context, text, icon='INFO'):
    """Draw a label that wraps onto multiple lines so it isn't clipped in a narrow panel.
    Wrapping accounts for wide CJK characters (Korean/Japanese)."""
    region = getattr(context, "region", None)
    width = getattr(region, "width", 220) if region else 220
    try:
        ui = context.preferences.system.ui_scale
    except Exception:
        ui = 1.0
    # budget in display-width units; ~6.5 px per half-width unit, minus icon/margin
    max_units = max(6, int((width - 28) / (6.5 * ui)))
    lines = _wrap_cjk(text, max_units)
    col = layout.column(align=True)
    for i, ln in enumerate(lines):
        col.label(text=ln, icon=(icon if i == 0 else 'BLANK1'))


def _set_uv_editor_image(context, image, uv_only=False):
    """Show `image` in Image/UV editors across all workspaces.
    If uv_only=True, only editors in UV mode are changed (not Texture Paint etc.)."""
    if image is None:
        return
    for screen in bpy.data.screens:
        for area in screen.areas:
            if area.type == 'IMAGE_EDITOR':
                for space in area.spaces:
                    if space.type == 'IMAGE_EDITOR':
                        if uv_only and getattr(space, "mode", 'UV') != 'UV':
                            continue
                        space.image = image


def _set_texture_paint_image(context, image):
    """Set the Texture Paint canvas image."""
    if image is None:
        return
    ts = getattr(context.scene, "tool_settings", None)
    ip = getattr(ts, "image_paint", None) if ts else None
    if ip is not None:
        try:
            ip.canvas = image
        except Exception:
            pass


def _has_visible_uv_editor(context):
    """True if the current window already shows an Image/UV editor."""
    win = getattr(context, "window", None)
    scr = getattr(win, "screen", None) if win else None
    if scr is None:
        return False
    return any(a.type == 'IMAGE_EDITOR' for a in scr.areas)


def _goto_uv_editing(context):
    """Switch to the 'UV Editing' workspace, unless a UV/Image editor is already visible."""
    if _has_visible_uv_editor(context):
        return
    ws = bpy.data.workspaces.get("UV Editing")
    if ws is None:
        # fallback: any workspace that contains an image editor
        for w in bpy.data.workspaces:
            if any(a.type == 'IMAGE_EDITOR' for scr in w.screens for a in scr.areas):
                ws = w
                break
    if ws is not None:
        try:
            context.window.workspace = ws
        except Exception:
            pass


def _on_bake_image_update(self, context):
    """When the panel image is chosen, apply it as the UV editor base image (UV Editing only)."""
    _set_uv_editor_image(context, self.vavatar_bake_image, uv_only=True)


# ===========================================================================
#  SHAPE KEY  —  ARKit 52 blendshapes
# ===========================================================================
ARKIT_52 = [
    # order follows the "Description of 52 blendshapes for iPhone face Tracking" document
    "browInnerUp",          # 1
    "browDownLeft",         # 2
    "browDownRight",        # 3
    "browOuterUpLeft",      # 4
    "browOuterUpRight",     # 5
    "eyeLookUpLeft",        # 6
    "eyeLookUpRight",       # 7
    "eyeLookDownLeft",      # 8
    "eyeLookDownRight",     # 9
    "eyeLookInLeft",        # 10
    "eyeLookInRight",       # 11
    "eyeLookOutLeft",       # 12
    "eyeLookOutRight",      # 13
    "eyeBlinkLeft",         # 14
    "eyeBlinkRight",        # 15
    "eyeSquintLeft",        # 16
    "eyeSquintRight",       # 17
    "eyeWideLeft",          # 18
    "eyeWideRight",         # 19
    "cheekPuff",            # 20
    "cheekSquintLeft",      # 21
    "cheekSquintRight",     # 22
    "noseSneerLeft",        # 23
    "noseSneerRight",       # 24
    "jawOpen",              # 25
    "jawForward",           # 26
    "jawLeft",              # 27
    "jawRight",             # 28
    "mouthFunnel",          # 29
    "mouthPucker",          # 30
    "mouthLeft",            # 31
    "mouthRight",           # 32
    "mouthRollUpper",       # 33
    "mouthRollLower",       # 34
    "mouthShrugUpper",      # 35
    "mouthShrugLower",      # 36
    "mouthClose",           # 37
    "mouthSmileLeft",       # 38
    "mouthSmileRight",      # 39
    "mouthFrownLeft",       # 40
    "mouthFrownRight",      # 41
    "mouthDimpleLeft",      # 42
    "mouthDimpleRight",     # 43
    "mouthUpperUpLeft",     # 44
    "mouthUpperUpRight",    # 45
    "mouthLowerDownLeft",   # 46
    "mouthLowerDownRight",  # 47
    "mouthPressLeft",       # 48
    "mouthPressRight",      # 49
    "mouthStretchLeft",     # 50
    "mouthStretchRight",    # 51
    "tongueOut",            # 52
]

SEP_TOP = "___ARKit___"
SEP_BOTTOM = "___EX___"

ARKIT_REF_URL = "https://hinzka.hatenablog.com/entry/2021/12/21/222635"
MMD_REF_URL = "https://docs.google.com/document/d/1dqzjyvJPa845Atbq5oUaWyt19OK_ia0m325nvYddHy4/edit"
SEP_MMD = "___MMD___"
SEP_END = "___END___"

# MMD morphs: union of the "VRChat MMD Visemes Guide" CHART 1 (55) and LIST 2,
# in CHART 1 order first, then LIST 2's additional (non-duplicate) morphs. 71 total.
MMD_MORPHS = [
    # === CHART 1 ===
    # EYES
    "まばたき", "笑い", "ウィンク", "ウィンク右", "ウィンク２", "ｳｨﾝｸ２右",
    "なごみ", "はぅ", "びっくり", "じと目", "ｷﾘｯ", "はちゅ目", "星目", "はぁと",
    "瞳小", "瞳縦潰れ", "光下", "恐ろしい子！", "ハイライト消", "映り込み消",
    "喜び", "わぉ?!", "なごみω", "悲しむ", "敵意",
    # MOUTH
    "あ", "い", "う", "え", "お", "あ２", "ん", "▲", "∧", "□", "ワ", "ω", "ω□",
    "にやり", "にやり２", "にっこり", "ぺろっ", "てへぺろ", "てへぺろ２",
    "口角上げ", "口角下げ", "口横広げ", "歯無し上", "歯無し下",
    # BROWS
    "真面目", "困る", "にこり", "怒り", "上", "下",
    # === LIST 2 (additional, not already in CHART 1) ===
    # EYEBROWS
    "左眉上", "左眉下", "右眉上", "右眉下",
    # EYES
    "なぬ！", "細目", "猫目",
    # MOUTH
    "はんっ！", "ぎゃーす", "がーん", "ギギギ",
    # MISC
    "頬染め", "涙目", "テレ", "目の幅涙", "瞳全消し",
]


class MESH_OT_arkit_create_from_list(bpy.types.Operator):
    bl_idname = "mesh.arkit_create_from_list"
    bl_label = "Shape Keys from List"
    bl_description = ("Create the ARKit 52 blendshapes as empty shape keys, "
                      "with ___ARKit___ below Basis and ___EX___ at the bottom")
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        return obj is not None and obj.type == 'MESH' and obj.mode == 'OBJECT'

    def execute(self, context):
        obj = _mesh_obj(context)
        if obj is None:
            self.report({'ERROR'}, "Active object is not a mesh.")
            return {'CANCELLED'}

        if obj.data.shape_keys is None:
            obj.shape_key_add(name="Basis", from_mix=False)

        existing = {kb.name for kb in obj.data.shape_keys.key_blocks}
        order = ["A", "I", "U", "E", "O", SEP_TOP] + ARKIT_52 + [SEP_BOTTOM]

        created = 0
        for name in order:
            if name in existing:
                continue
            obj.shape_key_add(name=name, from_mix=False)
            existing.add(name)
            created += 1

        self.report({'INFO'}, "Created %d shape key(s)." % created)
        return {'FINISHED'}


class MESH_OT_mmd_create_from_list(bpy.types.Operator):
    bl_idname = "mesh.mmd_create_from_list"
    bl_label = "MMD Shape Keys"
    bl_description = ("Create the standard MMD (Japanese) morphs as empty shape keys, "
                      "with ___MMD___ below Basis and ___EX___ at the bottom")
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        return obj is not None and obj.type == 'MESH' and obj.mode == 'OBJECT'

    def execute(self, context):
        obj = _mesh_obj(context)
        if obj is None:
            self.report({'ERROR'}, "Active object is not a mesh.")
            return {'CANCELLED'}

        if obj.data.shape_keys is None:
            obj.shape_key_add(name="Basis", from_mix=False)

        existing = {kb.name for kb in obj.data.shape_keys.key_blocks}
        order = [SEP_MMD] + MMD_MORPHS + [SEP_END]

        created = 0
        for name in order:
            if name in existing:
                continue
            obj.shape_key_add(name=name, from_mix=False)
            existing.add(name)
            created += 1

        self.report({'INFO'}, "Created %d MMD shape key(s)." % created)
        return {'FINISHED'}


class MESH_OT_arkit_split_lr(bpy.types.Operator):
    bl_idname = "mesh.arkit_split_lr"
    bl_label = "Split Shape Key L/R"
    bl_description = ("Split the active (symmetric) shape key into <name>Left and <name>Right. "
                      "Center vertices (|X| <= threshold) receive 0.5 on each side")
    bl_options = {'REGISTER', 'UNDO'}

    center_threshold: FloatProperty(
        name="Center Threshold",
        description="Vertices with |X| within this distance from 0 are treated as center (0.5 each side)",
        default=0.0001, min=0.0, max=1.0, precision=5,
    )
    invert_lr: BoolProperty(
        name="Invert L/R",
        description="By default +X is treated as Left. Enable if your model's Left is on -X",
        default=False,
    )
    keep_source: BoolProperty(
        name="Keep Source",
        description="Keep the original symmetric shape key after splitting",
        default=True,
    )

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        if obj is None or obj.type != 'MESH' or obj.mode != 'OBJECT':
            return False
        sk = obj.data.shape_keys
        return sk is not None and obj.active_shape_key_index > 0

    def execute(self, context):
        obj = _mesh_obj(context)
        if obj is None:
            self.report({'ERROR'}, "Active object is not a mesh.")
            return {'CANCELLED'}

        sk = obj.data.shape_keys
        if sk is None:
            self.report({'ERROR'}, "Mesh has no shape keys.")
            return {'CANCELLED'}

        idx = obj.active_shape_key_index
        if idx <= 0:
            self.report({'ERROR'}, "Select a non-Basis shape key to split.")
            return {'CANCELLED'}

        src = sk.key_blocks[idx]
        src_name = src.name
        basis = src.relative_key if src.relative_key is not None else sk.key_blocks[0]

        n = len(basis.data)
        if len(src.data) != n:
            self.report({'ERROR'}, "Vertex count mismatch between Basis and source key.")
            return {'CANCELLED'}

        basis_co = [basis.data[i].co.copy() for i in range(n)]
        src_co = [src.data[i].co.copy() for i in range(n)]

        thr = self.center_threshold
        pos_co = []
        neg_co = []
        for i in range(n):
            x = basis_co[i].x
            delta = src_co[i] - basis_co[i]
            if x > thr:
                fp, fn = 1.0, 0.0
            elif x < -thr:
                fp, fn = 0.0, 1.0
            else:
                fp, fn = 0.5, 0.5
            pos_co.append(basis_co[i] + delta * fp)
            neg_co.append(basis_co[i] + delta * fn)

        name_l = src_name + "Left"
        name_r = src_name + "Right"
        left_co, right_co = (pos_co, neg_co) if not self.invert_lr else (neg_co, pos_co)

        kb = obj.data.shape_keys.key_blocks

        def _get_or_add(name):
            # reuse an existing key in place (overwrite values, keep its position);
            # otherwise create a new one (appended at the end)
            existing = kb.get(name)
            if existing is not None and name != src_name:
                return existing
            return obj.shape_key_add(name=name, from_mix=False)

        kb_l = _get_or_add(name_l)
        for i in range(n):
            kb_l.data[i].co = left_co[i]

        kb_r = _get_or_add(name_r)
        for i in range(n):
            kb_r.data[i].co = right_co[i]

        if not self.keep_source:
            kb = obj.data.shape_keys.key_blocks
            if src_name in kb:
                obj.shape_key_remove(kb[src_name])

        kb = obj.data.shape_keys.key_blocks
        if name_l in kb:
            obj.active_shape_key_index = list(kb).index(kb[name_l])

        self.report({'INFO'}, "Split '%s' -> '%s' / '%s'." % (src_name, name_l, name_r))
        return {'FINISHED'}


# ===========================================================================
#  BAKE  —  projected UV + auto emit bake
# ===========================================================================
BAKE_EMPTY_UV = "Bake empty"
BAKE_IMG_NAME = "Bake"
BAKE_MAT_NAME = "bake_m"
BAKE_RES = 4096


def _get_or_create_bake_material(obj, image=None):
    """Ensure this object has its own bake material, make it active, and return it.
    Only a bake material ALREADY on this object is reused; otherwise a fresh one is
    created (bpy auto-uniquifies the name, so another object's bake_m is never touched).
    The material's Image Texture node (Color -> Principled Base Color) is set to `image`
    when one is given, on both creation and reuse."""
    mat = None
    for m in obj.data.materials:
        if m is not None and (m.name == BAKE_MAT_NAME or m.name.startswith(BAKE_MAT_NAME + ".")):
            mat = m
            break

    if mat is None:
        mat = bpy.data.materials.new(name=BAKE_MAT_NAME)
        mat.use_nodes = True
        nt = mat.node_tree
        nodes = nt.nodes
        links = nt.links
        bsdf = next((n for n in nodes if n.type == 'BSDF_PRINCIPLED'), None)
        tex = nodes.new('ShaderNodeTexImage')
        if bsdf is not None:
            tex.location = (bsdf.location.x - 320, bsdf.location.y)
            links.new(tex.outputs['Color'], bsdf.inputs['Base Color'])
        else:
            tex.location = (-320, 300)
        obj.data.materials.append(mat)

    # apply the selected image to the Image Texture node (create/find it if needed)
    if mat.use_nodes:
        nt = mat.node_tree
        tex = next((n for n in nt.nodes if n.type == 'TEX_IMAGE'), None)
        if tex is None:
            bsdf = next((n for n in nt.nodes if n.type == 'BSDF_PRINCIPLED'), None)
            tex = nt.nodes.new('ShaderNodeTexImage')
            if bsdf is not None:
                tex.location = (bsdf.location.x - 320, bsdf.location.y)
                nt.links.new(tex.outputs['Color'], bsdf.inputs['Base Color'])
        if image is not None:
            tex.image = image

    # make it the active material slot
    slots = obj.data.materials
    for i, m in enumerate(slots):
        if m == mat:
            obj.active_material_index = i
            break
    return mat


class OBJECT_OT_bake_make_projected_uv(bpy.types.Operator):
    bl_idname = "object.bake_make_projected_uv"
    bl_label = "Create Projected UV"
    bl_description = ("Create a new UV map named '%s' and apply Project From View "
                      "using the current viewport angle" % BAKE_EMPTY_UV)
    bl_options = {'REGISTER', 'UNDO'}

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        return obj is not None and obj.type == 'MESH'

    def execute(self, context):
        obj = _mesh_obj(context)
        if obj is None:
            self.report({'ERROR'}, "Select a mesh object first.")
            return {'CANCELLED'}

        me = obj.data

        # create/assign the "bake_m" material BEFORE making the UV map,
        # and apply the image selected in the panel to its Image Texture node
        sel_img = getattr(context.scene, "vavatar_bake_image", None)
        bake_mat = _get_or_create_bake_material(obj, image=sel_img)
        _set_uv_editor_image(context, sel_img, uv_only=True)

        uv = me.uv_layers.get(BAKE_EMPTY_UV)
        if uv is None:
            uv = me.uv_layers.new(name=BAKE_EMPTY_UV)
        me.uv_layers.active = uv

        area = next((a for a in context.screen.areas if a.type == 'VIEW_3D'), None)
        if area is None:
            self.report({'ERROR'}, "No 3D Viewport found to project from.")
            return {'CANCELLED'}
        region = next((r for r in area.regions if r.type == 'WINDOW'), None)

        prev_mode = obj.mode
        try:
            bpy.ops.object.mode_set(mode='EDIT')
            bpy.ops.mesh.select_all(action='SELECT')
            with context.temp_override(area=area, region=region, active_object=obj):
                bpy.ops.uv.project_from_view(
                    orthographic=False, camera_bounds=False,
                    correct_aspect=True, clip_to_bounds=False, scale_to_bounds=False,
                )
        except RuntimeError as e:
            self.report({'ERROR'}, "Project From View failed: %s" % str(e))
            bpy.ops.object.mode_set(mode=prev_mode)
            return {'CANCELLED'}

        bpy.ops.object.mode_set(mode='OBJECT')

        # make bake_m the material actually applied to the mesh (all faces use it)
        mats = list(obj.data.materials)
        if bake_mat in mats:
            bake_idx = mats.index(bake_mat)
            for poly in obj.data.polygons:
                poly.material_index = bake_idx

        _goto_uv_editing(context)
        self.report({'INFO'}, "Created '%s' and projected from view." % BAKE_EMPTY_UV)
        return {'FINISHED'}


_uv_enum_cache = []


def _uv_items(self, context):
    global _uv_enum_cache
    obj = context.active_object
    items = []
    if obj and obj.type == 'MESH' and obj.data.uv_layers:
        for i, uvl in enumerate(obj.data.uv_layers):
            items.append((uvl.name, uvl.name, "", i))
    if not items:
        items = [('NONE', "(no UV maps)", "", 0)]
    _uv_enum_cache = items
    return _uv_enum_cache


def _image_items(self, context):
    items = [(img.name, img.name, "") for img in bpy.data.images
             if img.name not in ('Render Result', 'Viewer Node')]
    if not items:
        items = [('NONE', "(no images)", "")]
    return items


class OBJECT_OT_auto_bake(bpy.types.Operator, ExportHelper):
    bl_idname = "object.auto_bake"
    bl_label = "Bake"
    bl_description = ("Bake the active material (Emit) to a 4096 32-bit image using the chosen UV map, "
                      "save as PNG, delete the 'Bake empty' UV, then restore the material")
    bl_options = {'REGISTER'}

    filename_ext = ".png"
    filter_glob: StringProperty(default="*.png", options={'HIDDEN'})

    @classmethod
    def poll(cls, context):
        obj = context.active_object
        return obj is not None and obj.type == 'MESH'

    def invoke(self, context, event):
        self.filepath = BAKE_IMG_NAME + ".png"
        return ExportHelper.invoke(self, context, event)

    def execute(self, context):
        obj = _mesh_obj(context)
        if obj is None:
            self.report({'ERROR'}, "Select a mesh object first.")
            return {'CANCELLED'}

        bake_uv = getattr(context.scene, "vavatar_bake_uv", 'NONE')
        if bake_uv in ('', 'NONE'):
            self.report({'ERROR'}, "Choose a UV map to bake.")
            return {'CANCELLED'}

        me = obj.data
        mat = obj.active_material
        if mat is None or not mat.use_nodes:
            self.report({'ERROR'}, "Active material with nodes is required.")
            return {'CANCELLED'}

        src_img = getattr(context.scene, "vavatar_bake_image", None)

        nt = mat.node_tree
        nodes = nt.nodes
        links = nt.links

        out = next((n for n in nodes if n.type == 'OUTPUT_MATERIAL'), None)
        if out is None:
            out = nodes.new('ShaderNodeOutputMaterial')
        bsdf = next((n for n in nodes if n.type == 'BSDF_PRINCIPLED'), None)

        created_nodes = []
        surf = out.inputs.get('Surface')
        orig_surf_from = surf.links[0].from_socket if surf.links else None

        # 5) disconnect Principled -> Output surface
        for l in list(surf.links):
            links.remove(l)

        # 6) UV Map node -> "Bake empty"
        uvnode = nodes.new('ShaderNodeUVMap')
        created_nodes.append(uvnode)
        uvnode.uv_map = BAKE_EMPTY_UV if BAKE_EMPTY_UV in me.uv_layers else bake_uv

        # 7) Image Texture node with chosen source image
        tex = nodes.new('ShaderNodeTexImage')
        created_nodes.append(tex)
        if src_img is not None:
            tex.image = src_img
        # 8) UV -> Image Texture.Vector
        links.new(uvnode.outputs['UV'], tex.inputs['Vector'])

        # 9) Emission + 10) Image color -> Emission
        emi = nodes.new('ShaderNodeEmission')
        created_nodes.append(emi)
        links.new(tex.outputs['Color'], emi.inputs['Color'])
        # 11) Emission -> Output Surface
        links.new(emi.outputs['Emission'], surf)

        # 12) create the Bake target image (New), transparent background
        bake_img = bpy.data.images.new(
            BAKE_IMG_NAME, width=BAKE_RES, height=BAKE_RES, float_buffer=True, alpha=True
        )
        bake_img.alpha_mode = 'STRAIGHT'
        bake_img.generated_color = (0.0, 0.0, 0.0, 0.0)  # transparent
        bake_tex = nodes.new('ShaderNodeTexImage')
        created_nodes.append(bake_tex)
        bake_tex.image = bake_img
        for n in nodes:
            n.select = False
        bake_tex.select = True
        nodes.active = bake_tex

        # step 4 -> chosen UV active for baking
        chosen = me.uv_layers.get(bake_uv)
        if chosen is not None:
            me.uv_layers.active = chosen

        def _restore_material():
            for n in created_nodes:
                try:
                    nodes.remove(n)
                except Exception:
                    pass
            tgt = out.inputs.get('Surface')
            src_sock = orig_surf_from
            if src_sock is None and bsdf is not None:
                src_sock = bsdf.outputs.get('BSDF')
            if src_sock is not None:
                try:
                    links.new(src_sock, tgt)
                except Exception:
                    pass

        # 13) Cycles + Emit bake
        prev_engine = context.scene.render.engine
        context.scene.render.engine = 'CYCLES'
        context.scene.cycles.bake_type = 'EMIT'
        try:
            bpy.ops.object.bake(type='EMIT', use_clear=False)
        except RuntimeError as e:
            self.report({'ERROR'}, "Bake failed: %s" % str(e))
            context.scene.render.engine = prev_engine
            _restore_material()
            return {'CANCELLED'}
        context.scene.render.engine = prev_engine

        # 14) save PNG (RGBA so transparency is kept, compression 100)
        bake_img.filepath_raw = self.filepath
        bake_img.file_format = 'PNG'
        scene_img = context.scene.render.image_settings
        prev_fmt = scene_img.file_format
        prev_comp = scene_img.compression
        prev_mode = scene_img.color_mode
        scene_img.file_format = 'PNG'
        scene_img.color_mode = 'RGBA'
        scene_img.compression = 100
        try:
            bake_img.save()
        except RuntimeError as e:
            self.report({'ERROR'}, "Save failed: %s" % str(e))
        scene_img.file_format = prev_fmt
        scene_img.compression = prev_comp
        scene_img.color_mode = prev_mode

        # 15) delete "Bake empty" UV
        be = me.uv_layers.get(BAKE_EMPTY_UV)
        if be is not None:
            me.uv_layers.remove(be)

        # 16) reset shading: remove added nodes + reconnect Principled -> Output
        _restore_material()

        # 17) set the material's Image Texture to the baked result, and show that
        #     image in the UV/Image editor and the Texture Paint canvas
        img_tex = next((n for n in mat.node_tree.nodes if n.type == 'TEX_IMAGE'), None)
        if img_tex is not None:
            img_tex.image = bake_img
            for n in mat.node_tree.nodes:
                n.select = False
            img_tex.select = True
            mat.node_tree.nodes.active = img_tex
        _set_uv_editor_image(context, bake_img)
        _set_texture_paint_image(context, bake_img)

        self.report({'INFO'}, "Baked and saved: %s" % self.filepath)
        return {'FINISHED'}


# ===========================================================================
#  Auto-update  (checks GitHub releases, downloads the .py, overwrites itself)
# ===========================================================================
# --- CONFIG: set these to your GitHub repository ---------------------------
UPDATE_OWNER = "Kanan-t"          # <-- your GitHub username
UPDATE_REPO = "Vavatar-tools"      # <-- your GitHub repository name
UPDATE_ASSET_NAME = "vavatar_tools_kn.py"   # the .py file attached to each release
# ---------------------------------------------------------------------------

# transient status shared with the UI
_update_state = {"checked": False, "available": False, "latest": None, "message": ""}


def _current_version():
    return tuple(bl_info.get("version", (0, 0, 0)))


def _parse_version(tag):
    # "v1.2.3" / "1.2.3" -> (1, 2, 3)
    nums = []
    for part in str(tag).lstrip("vV").replace("-", ".").split("."):
        if part.isdigit():
            nums.append(int(part))
    return tuple(nums) if nums else (0,)


def _fetch_latest_release():
    url = "https://api.github.com/repos/%s/%s/releases/latest" % (UPDATE_OWNER, UPDATE_REPO)
    req = urllib.request.Request(url, headers={"User-Agent": "avatar-tools-updater"})
    with urllib.request.urlopen(req, timeout=15) as resp:
        return json.loads(resp.read().decode("utf-8"))


def _asset_download_url(release):
    for asset in release.get("assets", []):
        if asset.get("name") == UPDATE_ASSET_NAME:
            return asset.get("browser_download_url")
    # fallback: raw file at the release tag
    tag = release.get("tag_name")
    if tag:
        return "https://raw.githubusercontent.com/%s/%s/%s/%s" % (
            UPDATE_OWNER, UPDATE_REPO, tag, UPDATE_ASSET_NAME)
    return None


def _check_impl():
    """Return (available, latest_version_tuple, message). Updates _update_state."""
    if not getattr(bpy.app, "online_access", True):
        _update_state.update(checked=True, available=False, latest=None,
                             message="Online access is off (Preferences > System > Network)")
        return _update_state
    try:
        rel = _fetch_latest_release()
    except Exception as e:
        _update_state.update(checked=True, available=False, latest=None,
                             message="Check failed: %s" % str(e))
        return _update_state
    latest = _parse_version(rel.get("tag_name", "0"))
    cur = _current_version()
    available = latest > cur
    _update_state.update(
        checked=True, available=available, latest=latest,
        message=("Update available: %s" % ".".join(map(str, latest))) if available
                else "Up to date (%s)" % ".".join(map(str, cur)),
    )
    _update_state["_release"] = rel
    return _update_state


class AVATAR_OT_check_update(bpy.types.Operator):
    bl_idname = "avatar.check_update"
    bl_label = "Check for Updates"
    bl_description = "Check the GitHub repository for a newer release"

    def execute(self, context):
        st = _check_impl()
        self.report({'INFO'} if not st["message"].startswith("Check failed") else {'WARNING'},
                    st["message"])
        return {'FINISHED'}


class AVATAR_OT_do_update(bpy.types.Operator):
    bl_idname = "avatar.do_update"
    bl_label = "Update Now"
    bl_description = "Download the latest release and overwrite this add-on file (restart Blender afterwards)"

    def execute(self, context):
        rel = _update_state.get("_release")
        if rel is None:
            self.report({'ERROR'}, "Run 'Check for Updates' first.")
            return {'CANCELLED'}
        url = _asset_download_url(rel)
        if not url:
            self.report({'ERROR'}, "Could not find a download URL for %s." % UPDATE_ASSET_NAME)
            return {'CANCELLED'}
        try:
            req = urllib.request.Request(url, headers={"User-Agent": "avatar-tools-updater"})
            with urllib.request.urlopen(req, timeout=60) as resp:
                content = resp.read()
        except Exception as e:
            self.report({'ERROR'}, "Download failed: %s" % str(e))
            return {'CANCELLED'}

        try:
            path = os.path.realpath(__file__)
            with open(path, "wb") as f:
                f.write(content)
        except Exception as e:
            self.report({'ERROR'}, "Could not write add-on file: %s" % str(e))
            return {'CANCELLED'}

        _update_state.update(available=False,
                             message="Updated. Please restart Blender to apply.")
        self.report({'INFO'}, "Updated. Restart Blender to apply the new version.")
        return {'FINISHED'}


class AVATARTOOLS_AddonPreferences(bpy.types.AddonPreferences):
    bl_idname = __name__

    auto_check: BoolProperty(
        name="Check for updates on startup",
        description="Quietly check GitHub for a newer release a few seconds after Blender starts, "
                    "then show a notice in the panel if one is available",
        default=True,
    )

    def draw(self, context):
        layout = self.layout
        layout.prop(self, "auto_check")
        row = layout.row(align=True)
        row.operator("avatar.check_update", icon='FILE_REFRESH')
        if _update_state.get("available"):
            row.operator("avatar.do_update", icon='IMPORT')
        if _update_state.get("checked"):
            icon = 'ERROR' if _update_state["message"].startswith("Check failed") else 'INFO'
            layout.label(text=_update_state["message"], icon=icon)
        layout.label(text="Repo: %s/%s" % (UPDATE_OWNER, UPDATE_REPO), icon='URL')


def _startup_check():
    try:
        prefs = bpy.context.preferences.addons[__name__].preferences
        if prefs and prefs.auto_check:
            _check_impl()
            if _update_state.get("available"):
                for w in bpy.context.window_manager.windows:
                    for area in w.screen.areas:
                        area.tag_redraw()
    except Exception:
        pass
    return None  # run once


# ===========================================================================
#  Panels  —  two tabs
# ===========================================================================
class VIEW3D_PT_vavatar_tools(bpy.types.Panel):
    bl_space_type = 'VIEW_3D'
    bl_region_type = 'UI'
    bl_category = "Vavatar_tools_KN"
    bl_label = "Vavatar_tools_KN"

    def draw(self, context):
        layout = self.layout

        # update notification banner — shown automatically when a newer version was found
        if _update_state.get("available"):
            latest = ".".join(str(x) for x in (_update_state.get("latest") or ()))
            box = layout.box()
            col = box.column(align=True)
            col.alert = True
            col.label(text="New version %s available!" % latest, icon='INFO')
            col.operator("avatar.do_update", text="Update Now", icon='IMPORT')

        obj = context.active_object
        if obj is None or obj.type != 'MESH':
            layout.label(text="Select a mesh object.", icon='INFO')
            return

        # --- Shape Key section ---
        sk_box = layout.box()
        sk_box.label(text="Shape Key", icon='SHAPEKEY_DATA')
        if obj.mode != 'OBJECT':
            sk_box.label(text="Only usable in Object Mode", icon='ERROR')
        # ARKit kit + reference link
        col = sk_box.column(align=True)
        col.operator("mesh.arkit_create_from_list", icon='SHAPEKEY_DATA')
        col.operator("wm.url_open", text="ARKit Reference", icon='URL').url = ARKIT_REF_URL

        # MMD kit + reference link
        col = sk_box.column(align=True)
        col.operator("mesh.mmd_create_from_list", icon='SHAPEKEY_DATA')
        col.operator("wm.url_open", text="MMD Reference", icon='URL').url = MMD_REF_URL

        # shared L/R split
        sk_box.separator()
        sk_box.operator("mesh.arkit_split_lr", icon='MOD_MIRROR')

        sk = obj.data.shape_keys
        if sk and obj.active_shape_key_index >= 0:
            akb = sk.key_blocks[obj.active_shape_key_index]
            sk_box.label(text="Active key: %s" % akb.name)
            if obj.active_shape_key_index == 0:
                sk_box.label(text="(Basis - pick another to split)", icon='INFO')

        # --- Bake section ---
        b_box = layout.box()
        b_box.label(text="Bake", icon='RENDER_STILL')
        # image select (applied to the bake_m Image Texture) — ABOVE Create Projected UV
        b_box.prop(context.scene, "vavatar_bake_image", text="Image")
        b_box.operator("object.bake_make_projected_uv", icon='UV_DATA')
        _wrap_label(b_box, context, "실행 전 오브젝트를 선택하고 뷰 화면을 맞춰주세요.")
        b_box.separator()
        # bake target UV select
        b_box.prop(context.scene, "vavatar_bake_uv", text="UV to Bake")
        b_box.operator("object.auto_bake", icon='RENDER_STILL')
        _wrap_label(b_box, context, "실행 전 UV와 이미지를 맞춰주세요.")

        # --- Update section ---
        u_box = layout.box()
        u_box.label(text="Update", icon='FILE_REFRESH')
        row = u_box.row(align=True)
        row.operator("avatar.check_update", icon='FILE_REFRESH')
        if _update_state.get("available"):
            row.operator("avatar.do_update", icon='IMPORT')
        if _update_state.get("checked"):
            icon = 'ERROR' if _update_state["message"].startswith("Check failed") else 'INFO'
            u_box.label(text=_update_state["message"], icon=icon)


classes = (
    MESH_OT_arkit_create_from_list,
    MESH_OT_mmd_create_from_list,
    MESH_OT_arkit_split_lr,
    OBJECT_OT_bake_make_projected_uv,
    OBJECT_OT_auto_bake,
    AVATAR_OT_check_update,
    AVATAR_OT_do_update,
    AVATARTOOLS_AddonPreferences,
    VIEW3D_PT_vavatar_tools,
)


def register():
    for c in classes:
        bpy.utils.register_class(c)
    bpy.types.Scene.vavatar_bake_uv = EnumProperty(name="Bake UV", items=_uv_items)
    bpy.types.Scene.vavatar_bake_image = PointerProperty(
        name="Bake Image", type=bpy.types.Image, update=_on_bake_image_update)
    try:
        bpy.app.timers.register(_startup_check, first_interval=3.0)
    except Exception:
        pass


def unregister():
    try:
        if bpy.app.timers.is_registered(_startup_check):
            bpy.app.timers.unregister(_startup_check)
    except Exception:
        pass
    for prop in ("vavatar_bake_uv", "vavatar_bake_image"):
        if hasattr(bpy.types.Scene, prop):
            delattr(bpy.types.Scene, prop)
    for c in reversed(classes):
        bpy.utils.unregister_class(c)


if __name__ == "__main__":
    register()
