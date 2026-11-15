# NES SMB3 Sprites in Super Mario All-Stars — Implementation Plan

## Overview

Allow users to supply an original NES *Super Mario Bros. 3* ROM (e.g., `smb3USARevA.nes` or `smb3.nes`) alongside the Super Mario All-Stars `.sfc` ROM to replace SMAS SMB3's 16-bit sprites (Mario, Luigi, enemies, items, objects, and world map sprites) with authentic 8-bit NES sprites and palettes at runtime.

Because SMAS SMB3 is a direct 6502-to-65816 port of NES SMB3 that preserved the original NES CHR bank structure, even/odd 8×16 tile numbering, and player pose indices, and because `Ppu` already supports sprite tile and palette overrides for co-op, this feature can be implemented cleanly without invasive changes to the emulator core.

---

## 1. Reverse-Engineering Findings & Technical Mapping

### 1.1 NES ROM Layout (`smb3USARevA.nes`)
* **Total Size**: 393,232 bytes (iNES Mapper 4 / MMC3).
  * **Header** (`0x00000–0x0000F`, 16 bytes): `4E 45 53 1A 10 10 40 00 ...` (16 × 16 KB PRG banks, 16 × 8 KB CHR banks).
  * **PRG-ROM** (`0x00010–0x4000F`, 256 KB): Thirty-two 8 KB MMC3 PRG banks (`0..31`).
  * **CHR-ROM** (`0x40010–0x6000F`, 128 KB): **128 1 KB MMC3 CHR banks** (`0x00..0x7F`), each containing **64 8×8 2bpp tiles** (16 bytes / 8 words per tile; `8,192` tiles total).

### 1.2 2bpp (NES) to 4bpp (SNES) Tile Conversion
* NES CHR tiles are **2bpp planar** (16 bytes per 8×8 tile: bytes `0..7` = bitplane 0, bytes `8..15` = bitplane 1).
* SNES OBJ tiles are **4bpp planar** (32 bytes / 16 `uint16_t` words per 8×8 tile: words `0..7` = bitplanes 0 & 1 interleaved by row; words `8..15` = bitplanes 2 & 3).
* Converting an 8×8 NES tile to SNES 4bpp format is an exact, lossless bitplane pack:
  ```cpp
  for (int y = 0; y < 8; y++) {
      outWords[y]     = uint16_t(nesTile[y]) | (uint16_t(nesTile[y + 8]) << 8);
      outWords[y + 8] = 0; // Bitplanes 2 & 3 are zero for 2bpp NES data
  }
  ```

### 1.3 Enemy, Item, and Object Sprites (1:1 Bank & Tile Mapping)
* **1:1 Linear Bank Formula (`$30–$37`)**: When Nintendo ported SMB3 to SMAS, they expanded the 128 1 KB NES CHR banks (`0x00..0x7F`, 128 KB at 2bpp) into 256 KB of 4bpp SNES CHR data across SMAS LoROM banks `$30–$37` (where each 1 KB NES bank became a `$0800`-byte / 2 KB 64-tile chunk):
  $$\text{NES 1 KB Bank } N = (\text{SMAS Bank} - \text{\$30}) \times 16 + \frac{\text{SMAS Offset} - \text{\$8000}}{\text{\$0800}}$$
  Conversely, NES 1 KB CHR bank `N` (`0x00..0x7F`) maps to:
  * **SMAS Bank**: `$30 + (N >> 4)`
  * **SMAS Address**: `$8000 + ((N & 0x0F) * 0x0800)`
* **Level Sprite CHR Loader (`$20:ADC0–$20:AEE0`)**: On level load, SMAS populates SNES OBJ VRAM `$6000–$7FFF` (512 tiles, `0x000..0x1FF`) in eight 64-tile (`$0800`-byte / `$0400`-word) slots indexed by `$1EBF` (`Level_Tileset`):
  * **Slot 0 (`$6000–$63FF`, tiles `0x000..0x03F`)**: Loaded from `$3F:B000` (common player/status/item tiles; `0x00..0x0B` are dynamically overwritten every NMI for Player 1, `0x0C..0x0F` at `$60C0` are animated per-frame from bank `$31` at `$20:F8A9`, and `$6100–$61BF` tiles `0x10..0x1B` are used for Player 2 in 2P Battle).
  * **Slot 1 (`$6400–$67FF`, tiles `0x040..0x07F`)**: Table at `$20:A909` (offset) / `$20:A969` (bank) — e.g. `$30:A000` $\leftrightarrow$ NES 1 KB bank `0x04` (powerups/items).
  * **Slot 2 (`$6800–$6BFF`, tiles `0x080..0x0BF`)**: Table at `$20:A999` / `$20:A9F9`.
  * **Slot 3 (`$6C00–$6FFF`, tiles `0x0C0..0x0FF`)**: Table at `$20:AA29` / `$20:AA89` — e.g. `$34:F800` $\leftrightarrow$ NES 1 KB bank `0x4F` (Goomba, Koopa, etc.).
  * **Slot 4 (`$7000–$73FF`, tiles `0x100..0x13F`)**: Table at `$20:AAB9` / `$20:AB19` — e.g. `$30:D800` $\leftrightarrow$ NES 1 KB bank `0x0B`.
  * **Slot 5 (`$7400–$77FF`, tiles `0x140..0x17F`)**: Table at `$20:AB49` / `$20:ABA9` — e.g. `$34:F000` $\leftrightarrow$ NES 1 KB bank `0x4E`.
  * **Slot 6 (`$7800–$7BFF`, tiles `0x180..0x1BF`)**: Table at `$20:ABD9` / `$20:AC39` — e.g. `$30:A800` $\leftrightarrow$ NES 1 KB bank `0x05`.
  * **Slot 7 (`$7C00–$7FFF`, tiles `0x1C0..0x1FF`)**: Loaded from `$3F:B800` (or dynamically swapped via `$023E–$0242` at `$29:EB69–$29:EBA1`).
* **Preserved 8×16 Even/Odd Tile Indexing (`$22:CBA0–$22:CC60`)**:
  * NES SMB3 used 8×16 sprite mode (`PPUCTRL` bit 5 = 1), where each 8×16 column uses even tile `T` (top 8×8) and odd tile `T + 1` (bottom 8×8).
  * SMAS SMB3 uses 8×8 / 16×16 SNES sprites (`OBSEL = 0x03`, `objBase = $6000`), and its enemy/object draw routine (`$22:CBA0`) reads the **same even tile numbers** from `$21:B744,X` and emits four 8×8 sprites (`T`, `T + 1` on the left; `T2`, `T2 + 1` on the right, swapping top/bottom when vertically flipped).
  * Because both the 64-tile bank offsets and the even/odd tile layout match, replacing a 64-tile SMAS chunk with the converted 64-tile NES 1 KB CHR bank automatically places every enemy/object tile at the exact index SMAS OAM expects.

### 1.4 Player Sprites (Mario & Luigi)
* **NES Player Sprite Tables (`smb3USARevA.nes` PRG Bank 29, `0x3AC51–0x3AEAD`)**:
  * **Pose Tile Table (`0x3AC51–0x3AE48`, 504 bytes = 84 frames × 6 bytes)**: For each pose (`0..83`), gives the 6 8×16 tile numbers (`top-L, top-M, top-R, bot-L, bot-M, bot-R`) inside the active 1 KB player CHR bank (`R2`, tiles `0x00..0x3F`, with `0xF1` = empty/blank tile).
  * **Frame-Group Offset Table (`0x3AE37` / `0x3AE49`, 84 bytes)**: Maps `frame` (`0..83`) to CHR bank offset `0, 1, 2, 3` within the suit's 4-bank group.
  * **Suit Base CHR Bank Table (`0x3AE98`, 7 bytes)**: `50 54 54 00 50 40 44` for suits `0..6` (Small = `0x50`, Super = `0x54`, Fire = `0x54`, Raccoon = `0x00`, Frog = `0x50`, Tanooki = `0x40`, Hammer = `0x44`).
    $$\text{NES Player 1 KB CHR Bank} = \text{SuitBase}[\text{suit}] + \text{FrameGroup}[\text{frame}]$$
* **SMAS Player Sprite Streaming (`$20:E238` $\to$ `$22:E000` & NMI `$20:F668`)**:
  * SMAS always draws Player 1 as a 24×32 grid of twelve 8×8 sprites using fixed VRAM tiles `0x00..0x0B` (`$6000–$60BF`):
    * OAM `+6..+11` (`$0818–$082F`): Top 16×16 rows (tiles `0x00..0x05`, 3 columns × 2 rows).
    * OAM `+0..+5` (`$0800–$0817`): Bottom 16×16 rows (tiles `0x06..0x0B`, 3 columns × 2 rows).
  * `$22:E000` reads WRAM `$00BB` (`Player_Suit`, `0..6`) and `$00BC` (`Player_Frame`, `0..167`, where `frame >> 1` is `0..83`), looks up the **identical 84 × 6 pose table at `$21:8777`** (where each byte is `NES_byte & 0xFE`, and `0x10` replaces `0xF1` for blank), and writes six ROM source addresses to WRAM `$0220, $0222, $0224, $0226, $0228, $022A` + bank `$0238`.
  * During NMI (`$20:F668`), SMAS DMAs six 64-byte (2-tile) chunks from those pointers into VRAM `$6000, $6020, $6040, $6060, $6080, $60A0` (and `$022C..$0236` / bank `$0239` into `$6100..$61BF` for Player 2).

### 1.5 World Map, Bonus Games & 2P Battle Sprites
* **World Map (`$20:8370` & `$20:91E0`)**: DMAs `$4000` bytes (512 tiles) from `$38:C000–$38:FFFF` to VRAM `$6000–$7FFF`. Map sprites (Mario/Luigi map icons, Hammer Bro, Airship, Tank, Canoe, Help bubble, etc.) map to NES 1 KB CHR banks `0x10..0x17` and `0x58..0x5B`.
* **Spade / N-Spade Bonus (`$20:9020`)**: DMAs `$4000` bytes from `$39:8000–$39:BFFF` to VRAM `$6000–$7FFF`.
* **2P Mario Bros. Battle (`$20:CC00`)**: DMAs `$2800` bytes from `$0C:D000` to `$6000`, plus `$3D:E000` (`$1000` bytes to `$6000`) and `$3D:F000` (`$0800` bytes to `$6C00`).

---

## 2. Implementation Architecture

### Phase 1: NES ROM Discovery, Validation & CHR Conversion
**Files**: `src/game/smb3.h`, `src/game/smb3.cpp`, `src/app/app.cpp`, `src/app/config.h`, `src/app/config.cpp`

1. **ROM Discovery**:
   * Add `std::string nesRomPath` and `bool nesSprites = false` to `AppConfig` (`src/app/config.h` / `src/app/config.cpp`).
   * On startup or when loading the `.sfc` ROM in `src/app/app.cpp`, search for an NES SMB3 ROM in:
     1. Explicit path from config / CLI (`--nes-rom <path>`).
     2. Same directory as the `.sfc` ROM (`smb3USARevA.nes`, `smb3.nes`, `Super Mario Bros. 3 (USA) (Rev 1).nes`, `Super Mario Bros. 3 (USA).nes`).
     3. Current working directory.
2. **Validation (`Smb3NesRom`)**:
   * Verify file size is `393,232` bytes (16-byte iNES header + 256 KB PRG + 128 KB CHR) and header starts with `4E 45 53 1A 10 10`.
   * Store the 256 KB PRG-ROM (`nesPrg_[262144]`) and pre-convert all `8,192` 8×8 2bpp tiles in the 128 KB CHR-ROM into a 256 KB 4bpp SNES word buffer (`nesChr4bpp_[128 * 1024]` `uint16_t` words = 128 1 KB banks × 64 tiles × 16 words).

### Phase 2: Player Sprite Reconstruction (P1, P2 & Co-op)
**Files**: `src/game/smb3.h`, `src/game/smb3.cpp`, `src/game/smb3coop.cpp`

1. **Helper `buildNesPlayerTiles(uint8_t suit, uint8_t frame, uint16_t out192[192])`**:
   * Given `suit` (`$00BB`, `0..6`) and `frame` (`$00BC`, `0..167`, `poseIdx = frame >> 1`):
     * Read the 7-byte suit base table from NES PRG `0x3AE98` (`{0x50, 0x54, 0x54, 0x00, 0x50, 0x40, 0x44}`) and frame-group offset from NES PRG `0x3AE49[poseIdx]` (handling special frames where SMAS `$21:896F[poseIdx] >= 0x20`, such as Kuribo's Shoe / death pose).
     * Compute `chrBank1K = suitBase + frameGroup`.
     * Read the 6 tile bytes `t[0..5]` from NES PRG `0x3AC51 + poseIdx * 6` (or SMAS `$21:8777 + poseIdx * 6`).
     * For each of the 6 8×16 slots (`i = 0..5`):
       * If `t[i] == 0xF1` (or `0x10` in SMAS): zero out the 32 words (`out192[i * 32 .. i * 32 + 31] = 0`).
       * Otherwise, `tileIdx = t[i] & 0x3E`, and copy 2 consecutive 8×8 tiles (`tileIdx` and `tileIdx + 1`, 32 words total) from `nesChr4bpp_[chrBank1K * 1024 + tileIdx * 16]` into `out192[i * 32]`.
2. **Player 1 VRAM Injection (`Smb3Game::postFrame`)**:
   * When `nesSprites` is enabled and SMB3 is in a level (`isInLevel()`), write the 192 words for Player 1 (`suit = r8(0x00BB)`, `frame = r8(0x00BC)`) directly into `bus_.ppu().vramData()` at `$6000..$60BF`.
3. **Co-op Player 2 Integration (`Smb3Game::coopOam` in `src/game/smb3coop.cpp`)**:
   * In `coopOam()`, when `nesSprites` is enabled, populate `p2Tiles_[192]` via `buildNesPlayerTiles(p2_.suit, p2_.frame, p2Tiles_)` instead of reading SMAS ROM bank `$0238` pointers `$0220..$022A`.
   * Because `ppu.objAltTiles = p2Tiles_` (`objAltTileBase = 0x6000, objAltTileWords = 192`) is already wired into `Ppu::renderSprites()`, Player 2 automatically renders with NES sprites and independent pose/suit in co-op.

### Phase 3: Enemy, Item, Object & Map Sprite Replacement
**Files**: `src/game/smb3.h`, `src/game/smb3.cpp`

1. **64-Tile Bank Mapping for Level Sprites (`$6000–$7FFF`)**:
   * Provide a clean mapping from SMAS 64-tile (`$0800`-byte) sprite CHR chunks to NES 1 KB CHR banks (`0x00..0x7F`):
     * For standard banks in `$30–$36`, use the linear formula `nesBank = (smasBank - 0x30) * 16 + (smasAddr - 0x8000) / 0x0800` (with overrides for the few reorganized chunks such as `$34:A000` and `$32:D800`).
     * For common tiles in Slot 0 (`$6000–$63FF`, tiles `0x10..0x3F` from `$3F:B000`) and Slot 7 (`$7C00–$7FFF`, tiles `0x1C0..0x1FF` from `$3F:B800`), build a composite 64-tile NES table from NES common sprite banks `0x58`, `0x59`, `0x7E`, and `0x04`.
     * For per-frame animated sprite tiles at `$60C0–$60FF` (tiles `0x0C..0x0F`, DMA'd at `$20:F8A9` from bank `$31`), map the 4 animated tiles to their NES equivalents.
2. **World Map Sprites (`$6000–$7FFF` when on World Map)**:
   * Map the active world map sprite tiles in `$38:C000` (Small/Super/Suit Mario & Luigi map icons, Hammer Bro, Airship, Canoe, Help bubble, Fortress lock/boom) to NES 1 KB CHR banks `0x10..0x17` and `0x58..0x5B`.
3. **Delivery Mechanism (VRAM Sync in `postFrame()` vs. Reversible ROM Bank Swap)**:
   * **Recommended approach**: Maintain a shadow copy or apply a reversible in-memory patch to the sprite-only regions of SMAS banks `$30–$3F` when `nesSprites` is toggled on (backing up original SMAS bytes to restore immediately when toggled off), **plus** per-frame player tile injection at `$6000–$60BF` in `postFrame()`.
   * If any 64-tile chunk in `$30–$36` is shared with BG tiles, override OBJ VRAM `$6000–$7FFF` in `postFrame()` (or via a dedicated `Ppu` OBJ VRAM override pointer) so BG tiles remain untouched.

### Phase 4: NES Sprite Palettes (BGR555 Conversion & CGRAM / PPU Override)
**Files**: `src/game/smb3.cpp`, `src/game/smb3coop.cpp`, `src/core/ppu.h`, `src/core/ppu.cpp`

1. **Why Palette Mapping Is Required**:
   * NES 2bpp tiles only use color indices `1..3` of each sprite palette (index `0` is transparent).
   * SMAS 4bpp palettes place dark outlines or mid-tones at indices `1..3` and spread skin/suit colors across indices `4..15`. Without NES palette colors in slots `1..3`, 2bpp NES tiles will render with wrong colors.
2. **NES Master Palette LUT (64 NES Colors $\to$ 15-bit SNES BGR555)**:
   * Include a 64-entry `uint16_t kNesPaletteBgr555[64]` table converting NES `$00..$3F` hue/luminance indices to 15-bit SNES BGR555.
3. **Player & Enemy Sprite Palettes**:
   * **Player 1 (OBJ Palette 0 / CGRAM `128..131`) & Player 2 / Co-op (`objAltColors` / `objAlt2Colors`)**:
     * Populate indices `1..3` with NES SMB3's exact 3-color suit palettes from NES PRG:
       * Mario Small/Super/Raccoon/Tanooki: NES `($16, $36, $0F)` (red/orange, peach, black)
       * Luigi Small/Super/Raccoon/Tanooki: NES `($2A, $36, $0F)` (green, peach, black)
       * Fire Mario/Luigi: NES `($27, $36, $16)` (orange, peach, red)
       * Frog Mario/Luigi: NES `($2A, $36, $0F)` / `($16, $36, $0F)`
       * Hammer Mario/Luigi: NES `($30, $27, $0F)` (white, peach/orange, black)
   * **Enemies & Items (OBJ Palettes 1–7 / CGRAM `144..255`)**:
     * In `postFrame()` (or via `Ppu::objPalOverride`), set indices `1..3` of the active SNES OBJ palettes to the corresponding NES 3-color sprite palettes for the current level/map tileset (`$1EBF`), while leaving BG palettes (`CGRAM 0..127`) untouched.

### Phase 5: Configuration, CLI & In-Game Menu Toggle
**Files**: `src/app/config.h`, `src/app/config.cpp`, `src/app/menu.cpp`, `src/app/app.cpp`

1. **Config & Persistence (`smas.cfg`)**:
   * Save and load `nes_sprites = 0/1` and optional `nes_rom_path = ...` in `smas.cfg`.
2. **In-Game Pause / Options Menu (`src/app/menu.cpp`)**:
   * Add an **"NES Sprites (SMB3)"** toggle (`ON` / `OFF`, or `UNAVAILABLE (No NES ROM)` if `smb3USARevA.nes` was not found).
   * Toggling the setting at runtime immediately switches player, enemy, item, and map sprites and palettes between SMAS 16-bit and NES 8-bit modes.

---

## 3. Verification & Testing Plan

1. **Headless Verification with `smastool`**:
   * Load `Super Mario All-Stars (USA).sfc` + `smb3USARevA.nes`.
   * Boot into SMB3 World 1 Map and World 1-1 (`warp 4 1 2`).
   * Verify VRAM `$6000–$60BF` contains converted 2bpp NES Mario tiles for Small, Super, Fire, Raccoon, Frog, Tanooki, and Hammer suits across walk/jump/skid/tail-wag poses.
   * Verify VRAM `$6400–$7FFF` contains converted 2bpp NES item and enemy tiles (Mushroom, Super Leaf, Goomba `0xD8..0xDB`, Koopa Troopa, Piranha Plant, Goal Card).
   * Dump rendered screenshots (`screenshot`) in both SMAS mode and NES Sprite mode on the World Map, in 1-1, in a Fortress/Airship, and in 2P Co-op to verify pixel-accurate NES sprites and 3-color palettes.
2. **Co-op & Mode Transitions**:
   * Verify 2P Co-op renders both Mario (red NES palette) and Luigi (green NES palette) with independent NES poses (`p2Tiles_`) and Fire/Hammer/Frog suit palettes.
   * Verify toggling NES Sprites `ON` $\leftrightarrow$ `OFF` mid-level cleanly restores original SMAS 4bpp sprites and 16-color palettes.
