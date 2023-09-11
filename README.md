gvsave
======

Batteryless saving routines for GBA homebrew.

Saving on the GBA can be quite a pain.  This library auto-detects the flash chip of the cartridge,
and falls back to SRAM saving for emulator compatibility.

As a user, you will:

1. Configure the amount of save data you want
2. Initialize the library at run-time
3. Copy data from storage to RAM (reading the save)
4. Copy data from RAM to storage (writing the save)

NOTE: The last 16-bits of the save data will is used by the save routine! This is used to prevent
data corruption if the player shuts off the device in the middle of a save.

Flash carts will erase entire sectors at a time, so gvsave uses a backup sector to prevent data
loss. The last 16-bits tracks which sector is authoritative.

Assembler
---------

The gvsave project is designed to be used with the [gvasm](https://github.com/velipso/gvasm)
assembler.

It should work with gvasm 2.3.0+.

Demo
----

The demo shows some diagnostic information, allows you to save/load data, and verifies the data was
correctly loaded.

You can compile it via:

```
gvasm make main.gvasm
```

Or you can download the ROM directly, here:

[Demo](./main.gba)

How to Use
----------

You will need the files in `src/`:

* `src/config.gvasm` -- configuration
* `src/saveInit.gvasm` -- auto-detects the save type
* `src/saveCopy.gvasm` -- copies data from/to storage

### Configuration

You will need to provide three pieces of information in `src/config.gvasm`:

#### 1. `SAVE_SIZE`

This should be a power of 2, and be at least 4 bytes.  The last 2 bytes of the save will be used by
gvsave, so you should reserve space for it.

You should probably not go larger than 32KB (`0x8000`), in order to maximize compatibility with
emulators.

#### 2. `SAVE_ADDR`

This is the location in RAM that the save will transfer from/to.

When you call `saveCopy.fromStorage`, data will copy from storage to this location in RAM.

When you call `saveCopy.toStorage`, data from this location in RAM will be transfered to storage.

#### 3. `Save` data location

You need to provide a location in RAM to store the `Save` struct. By default, it is stored at the
beginning of IWRAM (`0x03000000`):

```
.struct Save = 0x03000000
  ...
.end
```

### Copying code to IWRAM

The two files `src/saveInit.gvasm` and `src/saveCopy.gvasm` should be executed from IWRAM.

They **cannot** be executed from ROM, or it will interfere with saving to cart.

This is a little complicated, because you will need to copy the code to IWRAM when your program
starts.

You will probably have other code that needs to be installed into IWRAM too (for example, sound
routines), so this can happen at the same time.

You will see this happen in the demo.

### Initialization

During startup, you should call `saveInit`:

```
ldr   r0, =saveInit+1
bl    bx_r0

...

.begin bx_r0
  .thumb
  bx    r0
.end
```

Note, you need to add `+1` because the routine is written in Thumb.

This will write to `Save.failCode`, but since it will fallback to SRAM, there isn't really a
failure state.

You can detect which type of save was detected via `Save.type`, where 0 is SRAM, and non-zero is
a flash cart.

### Loading saved data

In order to load data from storage to RAM, call:

```
// copy save from storage to EWRAM
ldr   r0, =saveCopy.fromStorage+1
bl    bx_r0
```

### Saving data

In order to save data to storage, call:

```
// copy save from EWRAM to storage
ldr   r0, =saveCopy.toStorage+1
bl    bx_r0
```

Note that this will disable interrupts, so you will want to fade out music, pause animations, etc,
prior to calling this function.

This can take a couple seconds to finish depending on the cart and size of the save.

It will return a status in `r0`, where `0` is success, and some other code is a failure. This
failure code will also be stored in `Save.failCode`.
