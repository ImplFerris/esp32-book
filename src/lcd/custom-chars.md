## Custom Characters

Besides the supported characters, you can create your own custom ones, like smileys or heart symbols. The module includes 64 bytes of Character Generator RAM (CGRAM), allowing up to 8 custom characters. 
 
Each character is an 8x8 grid, where each row is represented by a single 8-bit value (`u8`). This makes it 8 bytes per character (8 rows × 1 byte per row). That's why, with a total of 64 bytes, you can only store up to 8 custom characters (8 chars × 8 bytes = 64 bytes).

<img style="display: block; margin: auto;width:400px;" alt="custom characters grid" src="./images/custom-character-grid-bits.jpg"/>

Note: If you recall, in our LCD module, each character is represented as a 5x8 grid. But wait, didn't we say we need an 8x8 grid for the characters? Yes, that's correct-we need 8 x 8 (8 bytes) memory, but we only use 5 bits in each row. The 3 high-order bits in each row are left as zeros.

