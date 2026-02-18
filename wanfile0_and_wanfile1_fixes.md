# Context

I have extracted WanFile0 and WanFile1 similar, however, some frames are not being rendered, even though evidence using the `skytemple_files` code it rendered that frame correctly, might be some issue compressing the files?

## ROM Runtime Behavior:

1. At dungeon init, `FUN_022bd82c` loads file 292 from effect.bin into VRAM — this populates the **tile image data and palette**. File 292 is then discarded (it's a temporary loader).

2. File 0 is loaded and stored at `+0x2788` (state 0 shared sprite). File 1 stored at `+0x278C` (state 1). These contain **frame definitions and animation sequences only** — no image data. They reference tiles already in VRAM from step 1.

3. When a type 1 effect (WanFile0) fires, the ROM reads `animation_index` to pick which sequence from file 0's animation table. It reads `palette_index` to select which 16-color sub-palette row within the 256-entry palette. The OAM hardware renders 8bpp pixels against that palette row.

4. Multiple effects share the same animation but with different `palette_index` values — e.g., Water Gun and Mud Shot use the same hit circle animation but different color palettes.

**Assidion's Offline Approach (test.py):**

1. Parse file 292 fully using `FileType.WAN.EFFECT.deserialize()` — this gives the complete WAN with `imgData`, `customPalette`, `frameData` (empty), `animData` (empty).

2. Parse file 0 (or whichever `index_anim` is requested) — this gives a WAN with `frameData` and `animData` but no `imgData`.

3. **Manually construct a hybrid WAN object:**
   - `imgData` ← from file 292 (tile pixels)
   - `customPalette` ← from file 292 (colors)
   - `frameData` ← from file 0 (meta-frame piece definitions)
   - `animData` ← from file 0 (animation sequences)

4. Apply palette offset: loop through all `frameData` pieces and do `useConfig.paletteIndex += palette_index_value`. This shifts which 16-color sub-palette each piece uses within the 256-entry row.

5. Export using skytemple_files' standard WAN sheet exporter, which handles the 8bpp tile rendering internally.

**Key insight:** skytemple_files' deserializer and renderer already understand the WAN format natively — the 128-byte block alignment, 8bpp tile lookup, sub-palette indexing. Assidion only needed to stitch two files together and adjust palette indices. Our Rust scraper had to replicate all of that tile addressing logic manually.

# More Context about WanFile0

## Discord Enquires

### First one made by me (March 3rd 2025)

```
# Mapping moves to animation files

Me: I'm using the skytemple-files library to extract sprites and move effects from the rom and have successfully done so with the help of the documentation. I am now trying to map the moves to their respective animation files.

I tried to find the relationship through Move Animation Table(0x022C9064 NA) and Effect Animation Table (0x022CC52C NA) through functions like GetMoveAnimation(move_id) , GetEffectAnimation(effect_id), PlayMoveAnimation() and I wanted to verify my findings via the memory watcher and a script but it seems I was extracting assembly opcodes from the memory address rather then effect animation ids I was looking for. Think I'm trying to extract from the non-running ROM file while the correct data only exists when the game is running in memory.

Anyone know how to properly access these data structures from the ROM?  or have any information adjacent to this?

Do you have any ideas on how the anim effect id's match with map to the actual sprites in EFFECT/effect.bin.

Do you know how the game determines which specific file in effect.bin to use for a given effect_id? I see there's a file_index field in the effect_animation struct - is this directly referencing a file index in the effect.bin archive? From testing doesn't seem like it.

assidion: did you convert the hex value to a decimal value? it definitely is as it is passed to some of the DirectoryFileMngr functions as the file number
there is an easier way to do it though, each effect in Lists -> Animations -> General has a file id and also a value called unk2 which is the index in the file (don't know why it's still labeled like that)

Me: Finally understand it now the file index contains the dir of file in effect.bin and inside it there is a animation_index(offset 0xC in effect_animation) to tell you which file in that dir to use. Do you happen to know if the file index is out of bounds like 0, but there is a animation_index where that points to?

Thank you very much for the help.

assidion: if the file index is 0, that means it points to a special file called wan file 0 which follows a different format from the others
don't know if you've managed to extract it, but audino helped me with that and i have it here

and this is a minor detail, but the game packs all those animations in files following the wan format, the script just extracts them into folders
https://projectpokemon.org/home/docs/mystery-dungeon-nds/wanwat-file-format-r50/
```

### Recent ROM Editting and Support thread (February 2026)

```
Tooby_Two
OP
Original Poster — 08/02/2026 17:34Sunday, 8 February 2026 17:34
Hi! I'm working on a PMD fangame and I would like to use the sprites in EoS, particularly the effect sprites, for my attacks. I'm looking at some of the attack animations in SkyTemple, particularly some of the hit effects, and I see that they're under WAN File 0. Somebody before me posted a ZIP file of the sprites from WAN File 0 (⁠support⁠Mapping moves to animation files), which holds all the effects for when Unk 1 is 14. But certain hit effects like Mud Shot's hit effect (shown in picture) is when Unk 1 is 1. So, I assume there's more hit effects that haven't been posted anywhere. I did look in the GitHub repository for this animation, but I didn't see the one for Mud Shot specifically. I tried to do research on how to decompile effect.bin to get the WAN file that holds these sprites. I assume I would use skytemple-rust/files to export the sprites, but I don't really know where to start for that. If anybody has any information on this, let me know!

assidion — 08/02/2026 18:14Sunday, 8 February 2026 18:14
iirc unk2 is the index of the actual effect, and unk1 has to do with what palette will be used
[18:15]Sunday, 8 February 2026 18:15
so you would look at id 45 in the zip file, and there should be an image with 01 at the end
[18:16]Sunday, 8 February 2026 18:16
(at least i remember something like that, not home so i can't check)

Tooby_Two
OP
Original Poster — 08/02/2026 18:17Sunday, 8 February 2026 18:17
In the zip file, ID 45 lines up with the Water Gun hit effect, and there's not a file with 01 at the end

assidion — 08/02/2026 18:23Sunday, 8 February 2026 18:23
ah, then the mud shot hit effect is a recolor of water gun's, the extractor just didn't get the other palettes

[18:24]Sunday, 8 February 2026 18:24
when i'm back i'll see if i can modify the script a bit to get those

Tooby_Two
OP
Original Poster — 08/02/2026 18:27Sunday, 8 February 2026 18:27
Thank you! I think you are right, the effect does look like a recolor of water gun's effect. If I could just get the palette for it I could swap the colors over with like an image editor or something. I'm not sure what script you're using to get them in the first place lol

assidion — 08/02/2026 20:57Sunday, 8 February 2026 20:57
i got the ones for unk1 = 1, here you go

0000.zip
251.68 KB
[20:58]Sunday, 8 February 2026 20:58
feel free to tell me if there's ones with a different unk1/palette that you need
09 February 2026

assidion
i got the ones for unk1 = 1, here you go

Tooby_Two
OP
Original Poster — 09/02/2026 03:47Monday, 9 February 2026 03:47
Thank you!! I don't want to be a bother to you if I come across other animations that I would like to have in the future lol. Is the script you used for extracting the sprites publicly available?

assidion — 09/02/2026 08:52Monday, 9 February 2026 08:52
if you want to do so yourself, download skytemple-files, place this python script and the folder inside EFFECT.zip into the main skytemple-files folder and run the script

EFFECT.zip
756.76 KB
Download
import os

from PIL import Image

import glob

from skytemple_files.common.types.file_types import FileType
test.py
test.py (7 KB)
:white_check_mark:
[08:53]Monday, 9 February 2026 08:53
change the 1 in the line useConfig.paletteIndex += 1 to the unk1 value you want to use
[08:55]Monday, 9 February 2026 08:55
the result will be in EFFECT/output/0000

Tooby_Two
OP
Original Poster — 09/02/2026 13:45Monday, 9 February 2026 13:45
Thank you so much!! This works great! I might go through all the WAN File 0 effects and rip all the ones that are used in-game. Thank you again!
```

#### Test.py

```py
import os

from PIL import Image

import glob
from skytemple_files.common.types.file_types import FileType
from skytemple_files.graphics.chara_wan.sheets import ExportSheets, ImportSheets
from skytemple_files.common.ppmdu_config.data import Pmd2Sprite, Pmd2Index
from skytemple_files.graphics.effect_wan.model import WanFile

import xml.etree.ElementTree as ET


def getAnimList(path):
    copy_of = {}
    anim_to_num = {}
    result_anims = []
    tree = ET.parse(os.path.join(path, "AnimData.xml"))
    root = tree.getroot()
    anims_node = root.find("Anims")
    for anim_node in anims_node.iter("Anim"):
        maybe_name = anim_node.find("Name")
        name = maybe_name.text

        index = -1
        index_node = anim_node.find("Index")
        if index_node is not None and index_node.text is not None:
            index = int(index_node.text)
            while len(result_anims) <= index:
                result_anims.append([])
            result_anims[index].append(name)
            anim_to_num[name] = index

        backref_node = anim_node.find("CopyOf")
        if backref_node is not None and backref_node.text is not None:
            backref = backref_node.text
            copy_of[name] = backref

    for key in copy_of:
        dest = key
        while dest in copy_of:
            dest = copy_of[dest]
        if dest in anim_to_num:
            dest_idx = anim_to_num[dest]
            result_anims[dest_idx].append(key)
        else:
            print("{0} has mising anim!".format(path))
    return result_anims


def getPmd2SpriteDef(path):
    anim_list = getAnimList(path)
    indices = {}
    for idx, anim_list in enumerate(anim_list):
        if len(anim_list) > 0:
            indices[idx] = Pmd2Index(idx, anim_list)

    return Pmd2Sprite(0, indices)


shadow_img = Image.open(
    os.path.join(os.path.dirname(__file__), "skytemple_files", "graphics", "chara_wan", "Shadow.png")
)


def ReadWriteCompareSize(base_path):
    for folder in os.listdir(base_path):
        full_path = os.path.join(base_path, folder)
        if os.path.isdir(full_path) and int(folder) <= 656:
            bytes_size = 0
            if os.path.exists(os.path.join(full_path, "AnimData.xml")):
                anim_list = getAnimList(full_path)
                wan = ImportSheets(full_path)

                bytes, _, _ = wan.sir0_serialize_parts()
                bytes_size = len(bytes)

                ExportSheets(full_path, shadow_img, wan, anim_list)
            with open(os.path.join(base_path, "sizes.txt"), "a+", encoding="utf-8") as txt:
                txt.write("{0},{1}\n".format(folder, str(bytes_size)))


def ReadEffectFromNumber(inDir, index):
    inFile = None
    for file in glob.glob(os.path.join(inDir, "effect_" + format(index, "04d"))):
        inFile = file
    if inFile == None:
        return None

    return get_move_effect(inFile)


def ReadScreenEffectFromNumber(inDir, index):
    inFile = None
    for file in glob.glob(os.path.join(inDir, "effect_" + format(index, "04d"))):
        inFile = file
    if inFile == None:
        return None

    return get_screen_effect(inFile)


def ConvertEffect0(baseDir, index_anim, effectDataImg):
    inDir = os.path.join(baseDir, "effect")
    outDir = os.path.join(baseDir, "output")

    effectDataAnim = ReadEffectFromNumber(inDir, index_anim)

    if effectDataAnim == None:
        return

    effectData = WanFile()
    effectData.imgType = effectDataImg.imgType
    effectData.imgData = effectDataImg.imgData
    effectData.frameData = effectDataAnim.frameData
    effectData.animData = effectDataAnim.animData
    effectData.customPalette = effectDataImg.customPalette
    effectData.is256Color = effectDataImg.is256Color
    effectData.paletteOffset = effectDataImg.paletteOffset
    for metaFrameData in effectData.frameData:
            for useConfig in metaFrameData:
                useConfig.paletteIndex += 1

    FileType.WAN.EFFECT.export_sheets(os.path.join(outDir, format(index_anim, "04d")), effectData)


def ConvertEffect(baseDir, index, baseEffect):
    inDir = os.path.join(baseDir, "effect")
    outDir = os.path.join(baseDir, "output")

    print(format(index, "04d"))

    effectData = ReadEffectFromNumber(inDir, index)

    if effectData == None:
        return

    mergeWithBasePalette(effectData, baseEffect.customPalette)

    FileType.WAN.EFFECT.export_sheets(os.path.join(outDir, format(index, "04d")), effectData)


def ConvertScreenEffect(baseDir, index):
    inDir = os.path.join(baseDir, "effect")
    outDir = os.path.join(baseDir, "output")

    print(format(index, "04d"))

    effectData = ReadScreenEffectFromNumber(inDir, index)

    if effectData == None:
        return

    FileType.SCREEN_FX.export_sheets(os.path.join(outDir, format(index, "04d")), effectData, 0)


def ConvertAllEffects(baseDir):
    effect_wan = ReadEffectFromNumber(os.path.join(baseDir, "effect"), 292)

    # exceptional case first file
    ConvertEffect0(baseDir, 0, effect_wan)

    for index in range(1, 290):
       try:
           ConvertEffect(baseDir, index, effect_wan)
       except Exception as ex:
           print(str(ex))

    for index in range(269, 290):
        try:
            ConvertScreenEffect(baseDir, index)
        except Exception as ex:
            print(str(ex))


# item_id = 4
# rom = NintendoDSRom.fromFile(r'eos.nds')


def get_sprite_chara(bin_pack, id):
    return FileType.WAN.CHARA.deserialize(bin_pack[id])


def get_status_effect(base_path):
    with open(base_path, "rb") as f:
        return FileType.SMA.deserialize(f.read())


def get_move_effect(base_path):
    with open(base_path, "rb") as f:
        return FileType.WAN.EFFECT.deserialize(f.read())


def get_screen_effect(base_path):
    with open(base_path, "rb") as f:
        return FileType.SCREEN_FX.deserialize(f.read())


def open_bin(rom, name):
    return FileType.BIN_PACK.deserialize(rom.getFileByName(name))


#sprite_def = getPmd2SpriteDef(r"sprite-0646-0002\from")
#wan = FileType.WAN.CHARA.import_sheets(r"sprite-0646-0002\from")

#wan = FileType.WAN.CHARA.delete_anim_from_wan(wan, 3)

#FileType.WAN.CHARA.export_sheets(r"sprite-0646-0002\to", wan, sprite_def)
# FileType.WAN.deserialize(FileType.WAN.CHARA.serialize(ground))
# monster = get_sprite_chara(open_bin(rom, "MONSTER/m_ground.bin"), item_id)
# with open(r'm_ground_0061_0x1363b0.wan', 'rb') as f:
#    m_ground = FileType.WAN.CHARA.WanFile(f.read())

#sma = get_status_effect(r"manpu_su.sma")

#for ii in range(16):
#   FileType.SMA.export_sheets(r"output", sma, ii)

ConvertAllEffects(r"./EFFECT")
```

For more information about the WanFile0 and WanFile1/ Type1 & Type2 look into research files:
* `Data Structures/effect_animation_info.md`
* `Data Structures/effect_context.md`
* `Systems/effect_lifecycle.md`

# When I have time I should fix the blank frame more
I can do this by using a python environment with skytemple_files API recreating it there and then porting it over to Rust. I might need to update the implementation however, I should only do it for the WanFile0. **Note** this implementation only handles WanFile0 not WanFile1 need to figure out that as well after I get the python env to work with WanFile0.
